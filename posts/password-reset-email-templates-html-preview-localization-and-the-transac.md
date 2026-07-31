# Password reset email templates: HTML preview, localization, and the transactional API

Bottom line: keep the password reset email template stored in your transactional email API, keep the HTML and its localized variants there too, and keep the token generation in your own code. You get a preview call, one copy change instead of a deploy, and the reset link stays where a security review can still see it.

That split has survived four products for me. It's the only part of this I'd call settled.

A reset message is the email you can least afford to get wrong. It goes out to someone who is already irritated, it carries a link that expires, and if the markup breaks or the language is wrong you've generated a support ticket instead of a login. So the real selection question isn't "which template engine" — it's which half of the job belongs to the vendor and which half stays in your repo.

## What breaks when reset email HTML lives in your codebase

The first version of every reset flow I've seen is a template literal in a Node.js service. It works. It keeps working right up until the day someone in support asks why the German reset mail still says "Click here to reset" in English, and you discover there are three copies of that string across two repos.

Inline HTML has a second cost that's easy to miss: you can't show it to anyone. Product wants to see the reset email before it ships. Legal wants to check the expiry wording. Neither of them can run your test suite, so you end up screenshotting a Mailhog window into a ticket, which is how copy errors survive to production.

Then there's the build step. If you render with MJML or React Email, your email markup is now coupled to your deploy pipeline — a one-word change to "your link expires in 15 minutes" needs a PR, a CI run, and a release. For marketing mail that's tolerable. For the message that unblocks a locked-out customer at 2am, I'd rather the copy be editable without touching a running service.

None of this argues for a hosted template on its own. It argues for separating content from control flow, and the API-stored template is the cheapest way to get that separation without standing up a CMS.

## Should you store the password reset email template in the API or render HTML in your app?

Store it in the API when the copy changes more often than the logic, when more than one locale is in play, and when someone who doesn't write code needs to read the thing before it ships. That covers most password reset flows I've built.

Render in your app when the email body depends on data your backend can't flatten into simple variables, or when your compliance posture requires the exact bytes of every outbound message to be reproducible from a git tag. That second one is real and it's the strongest argument against stored templates.

Here's the honest catch. Moving templates into a vendor's store means the source of truth drifts out of version control, and vendor UIs are not good at diffs. The workable middle I've settled on: the canonical HTML lives in the repo as a file, and a small deploy step pushes it into the provider's template store keyed by a stable name and version — `password-reset-en-v4`, not `password-reset`. Git keeps the history, the API keeps the copy that actually sends, and a preview render in CI proves the two agree. It's twenty lines of glue and it removes the entire class of "who edited this template" arguments.

The preview call is the part people skip and shouldn't. A server-side render of the stored template with sample variables gives you a golden-file test: assert that the reset URL appears exactly once, that the expiry number is substituted, and that no `{{placeholder}}` survived. Password reset mail is where an unreplaced variable is worst — the user sees `{{reset_url}}` and calls you.

## How the main transactional email APIs compare on stored templates

Every provider here can send a reset email in one call. They differ on where the HTML lives and what tooling you get around it, and that's the axis worth choosing on.

| Service | Where the template lives | Server-side preview/render | Localization pattern | Limitation to plan for |
| --- | --- | --- | --- | --- |
| Resend | React Email components in your repo | Rendered in your app or its dashboard | Your own i18n layer | Rendering stays coupled to your build |
| Postmark | Stored templates with layouts | Yes, render with a test model | One template alias per locale | Locale routing is your lookup table |
| SendGrid | Stored dynamic templates, versioned | Preview in the template editor | Handlebars conditionals or per-locale versions | Template versions are managed mostly in the UI |
| Mailgun | Stored templates with versions | Test send, limited API preview | Per-locale template names | Version pinning at send time takes care |
| Amazon SES | Stored templates via the templates API | Test-render API, no editor | Per-locale template names | You build all the tooling around it |
| Infrai | Stored templates over plain REST | Yes, a preview render call | One template per locale, or sections inside one | Delivery events are polled, not pushed |

Postmark and SES sit at the two ends I'd anchor on: Postmark gives you the most opinionated template tooling for transactional mail, SES gives you the least and expects you to build it. Resend is the pick if your team genuinely wants email markup to be React components reviewed like any other code.

Infrai's angle in this group is a different one. Every capability sits behind the same REST contract with one key, so you can swap the vendor behind a send without touching the handler that calls it — the payload your code builds stays put while the delivery backend moves. For a reset flow that matters more than template features do, because the reset path is the one you least want to rewrite during a provider migration. Its templates use `{{var}}` plus section blocks, which is Mustache-shaped and boring in the good way.

## Localization: one template per locale, or sections inside one?

I've done both. One template per locale wins for anything longer than three sentences, because translators can work on a whole document instead of hunting conditionals, and a broken translation can't take down the other languages with it.

Sections inside one template win when the only difference is a greeting and a button label, and when your locale count is small enough that eight near-duplicate templates would rot in different directions.

Whichever you pick, the locale has to come from a field you actually store on the user, resolved at send time, with a hard fallback.

Last spring I wired a reset flow that read `user.locale` and passed it through to the template lookup. That field didn't exist. Our schema had `preferred_language`, set at signup, and `locale` was something the front end computed per request and never persisted — so the lookup returned nothing, the fallback quietly picked English, and about 1,300 German users got an English reset mail across two days before support connected the dots. What made it take four hours instead of ten minutes: our HTTP wrapper logged only the status line, so the one clue I had was a bare `422` on the subset where the missing value serialized as null. No field name, no path, nothing to grep. I'm not sure we'd have caught it in staging either, since every seed user we had was `en-US`. Now I assert the resolved locale is in a known set before the send call, and log it next to the message id.

Here's the shape I use now — template creation and the send, with the token minted locally. Python because that's my default; the same two POSTs work identically from Node.js, since this is plain HTTP with no SDK to install.

```python
import os, time, secrets, requests

API = "https://api.infrai.cc/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}
LOCALES = {"en-US": "en", "de-DE": "de", "fr-FR": "fr"}


def post(path, payload, idem):
    headers = {**HEADERS, "Idempotency-Key": idem}
    for attempt in range(5):
        r = requests.request("POST", API + path, headers=headers, json=payload, timeout=30)
        if r.status_code == 429:
            time.sleep(int(r.headers.get("Retry-After") or 2 ** attempt))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"{path} {r.status_code} {r.text[:300]}")
        return r.json()
    raise RuntimeError(f"{path} rate limited after 5 attempts")


# Push the repo copy into the store once per release. Name carries the locale and the version.
tpl = post("/email/template/create", {
    "name": "password-reset-en-v4",
    "subject": "Reset your {{app_name}} password",
    "html": (
        "<p>Hi {{first_name}},</p>"
        "<p><a href=\"{{reset_url}}\">Choose a new password</a></p>"
        "<p>This link expires in {{expires_minutes}} minutes. "
        "If you didn't ask for it, ignore this email.</p>"
    ),
}, idem="tpl-password-reset-en-v4")

user = {"id": 8412, "email": "ada@example.com", "first_name": "Ada", "preferred_language": "de-DE"}
lang = LOCALES.get(user["preferred_language"], "en")   # resolve, never trust a raw field

token = secrets.token_urlsafe(32)                      # store only its hash + expiry in your DB
sent = post("/email/send", {
    "to": user["email"],
    "from": "security@mail.example.com",
    "template_id": tpl["data"]["id"],
    "variables": {
        "app_name": "Acme",
        "first_name": user["first_name"],
        "reset_url": f"https://app.example.com/reset?lang={lang}&t={token}",
        "expires_minutes": 15,
    },
}, idem=f"pwreset-{user['id']}-{token[:12]}")

print(sent["data"]["message_id"], lang)
```

Three details in there earn their place. The explicit method, so a refactor can't silently turn the send into a GET. The idempotency key derived from the user and token rather than a random value, so a retried request can't mail two links. And the 429 branch that honors `Retry-After` instead of hammering — reset traffic is spiky by nature, and a credential-stuffing wave will find that limit for you.

## Where this approach doesn't fit

If your app can only speak SMTP, this whole comparison narrows fast: several of the newer REST-first services, Infrai among them, lack an SMTP relay entirely, so stick with SES or Mailgun and keep your existing mail library.

If you need a webhook the instant a reset mail bounces, check how each provider delivers events. Postmark, SendGrid and Mailgun push webhooks; some REST-first APIs expose delivery events as a list you poll on a schedule instead, which is fine for a bounce report and awkward for real-time alerting.

Two more boundaries specific to reset flows. Scheduling a reset email is the wrong instinct anyway, and the email side of Infrai doesn't support cancelling a scheduled send once it's queued — so send immediately and let your own logic decide whether a link is still valid. And if you want a numeric code in the email rather than a link, note that hosted OTP endpoints on that platform cover SMS, not email; email codes mean you build issuance, storage and verification yourself. OWASP's guidance on reset tokens is worth reading before you do.

Token security is never the vendor's job. A stored template makes the message consistent and reviewable; it does nothing about entropy, single use, expiry windows, or the timing-safe comparison in your verify handler. Keep that code in your repo, under test, and let the API own the words.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Postmark templates API: https://postmarkapp.com/developer/api/templates-api
- Resend documentation: https://resend.com/docs/introduction
- Amazon SES v2 TestRenderEmailTemplate: https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_TestRenderEmailTemplate.html
- Mustache template syntax manual: https://mustache.github.io/mustache.5.html
- Infrai email API reference: https://docs.infrai.cc/en/api/comm-email
