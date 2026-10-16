# Trigger specifics

The trigger is what Converly's loader watches for on the user's website. Get the live catalogue with `converly triggers`; use a slug from `providers[]` as the flow's trigger.

## How detection works

The Converly snippet loads a per site config and watches the page for the configured form tool's submission signals (native events, embedded iframe messages, DOM signals, depending on the tool). No code changes to the form itself, no webhooks to set up for the browser detected tools. That means:

- The snippet MUST be on the page where the form lives.
- The flow must be PUBLISHED before anything is captured.
- Detection is per tool. Picking the right provider slug matters. `generic-form` covers hand coded `<form>` elements.

## Picking the right slug

Match what actually renders the form, not the brand of the website builder:

- A Typeform embedded on a Webflow site is `typeform`, not `webflow-forms`.
- A HubSpot form embedded anywhere is `hubspot-forms`.
- Webflow's own native form blocks are `webflow-forms`.
- A React app's hand rolled form is `generic-form`.

If you have repo or page access, inspect the form markup and decide yourself. Embed scripts (`js.hsforms.net`, `embed.typeform.com`) are the giveaway.

## Trigger types

- `form_submission`. About 80 providers, including Gravity Forms, Contact Form 7, WPForms, Elementor, Typeform, Jotform, Webflow, Wix, Squarespace, Framer, Unbounce, HubSpot, Mailchimp, Marketo, Tally, Fillout, ClickFunnels, and more. No filter is required. A flow with no filters fires for every submission of that type on the site, so offer narrowing when the site has more than one form.
- `chat_started` - LiveChat, Intercom, Drift, Tawk.to, HubSpot Chat and similar. Fires when a visitor starts a conversation.
- `meeting_booked` - Calendly, Acuity, Cal.com, OnceHub, SavvyCal style booking tools.
- Connection-required providers. Read each provider's `setup` block in the catalogue. `requires_connection: true` (Typeform, Jotform, Calendly, Acuity today) means the tool must be connected with `converly triggers connect <slug> --site <id>` BEFORE its flow properly works. `capture_without_connection` tells you honestly what happens if the user declines (for example count-only capture with no visitor details). `post_connect_user_steps` lists steps the USER must still do outside Converly (Acuity's pasted snippet); Converly cannot verify those, so setup is only proven when the first event arrives.
- `custom_event` - cannot be set up through this surface. Never offer it, and never improvise a custom-event workaround for an unsupported tool. If no listed provider fits, say so plainly.

## Filters

Write filters to `trigger_config.filters`. With no filters, every submission of the selected type counts, and the flow is valid and publishable.

**By page.** Every browser form tool supports page filters. For example, set `trigger_config.filters` to:

```json
{
  "match": "all",
  "list": [{ "field": "page", "op": "only", "values": ["/pricing", "/contact"] }]
}
```

The public format has `match` (`"all"` or `"any"`) and a `list` of rules. Each rule has `field`, `op` (`"only"` or `"except"`, where supported), and `values`. Page values are path strings, with `*` for a section once enabled. Other fields take objects such as `{ "id": "12" }` or `{ "name": "Demo Request" }`, where supported. Do not mix IDs and names in one rule. Use the trigger catalogue's `filters` to check which fields, operations and inputs the tool supports. The catalogue lists what the tool supports, not what this account can use yet. If the API refuses a filter as not yet available, fall back to an exact page path filter for a browser form tool or no filter and tell the user.

Wildcard page paths (values containing `*`), Gravity Forms form IDs, `except`, matching any of several rules, and page rules on connected tools are refused until enabled. A single rule with `match: "any"` is accepted if the rule itself is available.

**By page section, once enabled.** To count submissions on pages under `/services/`, set `trigger_config.filters` to:

```json
{
  "match": "all",
  "list": [{ "field": "page", "op": "only", "values": ["/services/*"] }]
}
```

**By Gravity Forms form ID.** Gravity Forms also supports including or excluding specific form IDs. Other browser form tools have no form filter. For Gravity Forms form 12, set `trigger_config.filters` to:

```json
{
  "match": "all",
  "list": [{ "field": "form", "op": "only", "values": [{ "id": "12" }] }]
}
```

**By a connected tool's form or event type.** After connecting the tool, get the account's real choices with `list_trigger_filter_options` or `converly triggers options <slug> --site <site_id>`. Offer those choices to the user, then put the selected ID in the matching field. For example, if Typeform returns a form with ID `abc123`, set `trigger_config.filters` to:

```json
{
  "match": "all",
  "list": [{ "field": "form", "op": "only", "values": [{ "id": "abc123" }] }]
}
```

Never invent IDs or names. Respect the result's `notes`. A field marked `available: false` means the choices could not be fetched, not that the account has none.

For `converly flows create`, the CLI provides these flags:

- `--pages /contact,/demo` includes those pages; `--except-pages /newsletter` excludes a page.
- `--form-ids 12,34` includes Gravity Forms IDs; `--except-form-ids 56` excludes an ID.
- `--match any` lets any rule match. Without it, all rules must match.
- `--filters-json` takes the public filters object above, including connected-tool filters. Do not combine it with the shortcut flags.

To change only an existing flow's filters, pass the public object directly to `flows update --filters-json`:

```sh
converly flows update <flow_id> --filters-json '{"match":"all","list":[{"field":"form","op":"only","values":[{"id":"abc123"}]}]}'
```

The legacy `pageFilter`, `pages` and `conditions` fields are deprecated and still accepted. Use `filters` for new writes.

## When nothing fires

Work through, in order:

1. `converly install status <site>`. The loader phones home on page load, so have any page of the site opened once, then re-run. Read `detection` precisely: `"confirmed"` means the loader runs on the site, look further down this list (and if `origin_authorized` is false, the domain is wrong, fix it with `sites update`). `"never_seen"` after a fresh page load means the snippet is probably not live (site builders need a republish), except for two heartbeat blind spots: Converly Webflow app installs and privacy-signal visitors, where only a real submission in `events list` proves the install. `"pending"` means the check itself couldn't run right now, try again shortly.
2. Site domain set and matching the real origin? `converly sites list` → `domain`. A staging URL or a different subdomain will be rejected.
3. Flow published? `converly flows get <id>` → `status` must be `published`, `publication_state` should be `in_sync`.
4. Right provider slug for what actually renders the form? (See "Picking the right slug".)
5. Internal traffic rule swallowing your tests? `converly rules list`.
6. Still nothing: `converly events list --status failed` and `converly events get` on anything that appears; the event detail carries per destination errors and user facing notices.
