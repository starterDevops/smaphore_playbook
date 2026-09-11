# The result contract

**Every playbook the Ops Portal drives must emit one JSON object per host,
between marker lines.** This is the interface between Ansible and the portal.
Get it right and a new automation needs no portal code at all.

```
OPSPORTAL_RESULT_BEGIN
{"host": "livestream5", "playbook": "update_ufw", "status": "success", ...}
OPSPORTAL_RESULT_END
```

---

## Why this exists

The portal shows a non-technical person what happened on each server. It cannot
do that by reading Ansible output — that output is written for engineers, its
format changes between modules, and parsing it by pattern-matching is guesswork
that breaks the first time a task name changes.

So the playbook states its own outcome, in a form a machine can read, and the
portal renders it.

**The rule that matters most:** a task Semaphore reports as successful, whose
output contains no result block, is **not** a success. It is `parse_failed`.
Silence and "nothing needed doing" are different answers, and a portal that
confuses them will one day show an approver an empty table and ask them to
approve it.

---

## Markers

New playbooks use `OPSPORTAL_RESULT_BEGIN` / `OPSPORTAL_RESULT_END`.

Four SSL playbooks predate this document and keep their own families. They are
production-tested, and renaming working markers on `main` — which is what
Semaphore executes — buys nothing, since the parser handles several patterns
either way.

| Playbook | Marker family |
|---|---|
| `update_ufw.yml` | `OPSPORTAL_RESULT_*` |
| *any new playbook* | `OPSPORTAL_RESULT_*` |
| `renew_apache_ssl.yml` | `SSL_RENEWAL_RESULT_*` *(legacy)* |
| `enable_apache_ssl.yml` | `SSL_ENABLE_RESULT_*` *(legacy)* |
| `inspect_ssl.yml` | `SSL_INSPECT_RESULT_*` *(legacy)* |
| `inspect_apache_ssl.yml` | `APACHE_INSPECT_RESULT_*` *(legacy)* |
| `deploy_ssl.yml` | `SSL_DEPLOY_RESULT_*` *(legacy)* |

`BEGIN` and `END` must belong to the **same** family — the parser uses a
backreference, so a truncated log cannot pair the start of one block with the
end of another.

---

## Required fields

| Field | Type | Meaning |
|---|---|---|
| `host` | string | `inventory_hostname`. Without it the portal cannot attribute the result to a server, and rejects the block |
| `playbook` | string | Stable identifier, e.g. `update_ufw`. Not the filename |
| `phase` | string | `preview` for a dry run, `execute` for a real one |
| `status` | string | From the vocabulary below |

## Status vocabulary

Keep to these. The portal maps each to plain-language text; an unrecognised
value renders raw, which is how jargon reaches a requester.

| `status` | Means | Shown to the user |
|---|---|---|
| `success` | Something was changed and it worked | "Updated" |
| `already_correct` | Checked, nothing needed doing | "Already correct — nothing changed" |
| `preview` | Dry run; describes what *would* happen | "Will be updated" |
| `not_applicable` | This automation does not apply to this host | "Not applicable to this server" |
| `failed` | Attempted and failed | "Could not be updated" |
| `skipped` | Deliberately passed over | "Skipped" |

`no_ssl` is a legacy alias for `not_applicable`, used by the SSL playbooks.
Do not use it in new work.

## Recommended fields

Optional, but each one removes a question a user would otherwise ask.

| Field | Type | Notes |
|---|---|---|
| `ticket_id` | string | Normalised — see the whitespace rule below |
| `request_id` | string | The portal's `req_uuid`, sent as `request_id` |
| `changed` | bool | Did anything actually change on this host |
| `summary` | string | **One plain-language sentence.** No jargon — a requester reads this |
| `skip_reasons` | list of strings | Why things were skipped. Plain language |
| `backup_path` | string | Where the pre-change state was saved |
| `rollback_available` | bool | Whether rollback is genuinely possible |
| `rollback_command` | string | Exact command. Only if `rollback_available` |
| `details` | object | Anything playbook-specific. The portal shows it verbatim to admins |

Put automation-specific data in `details` rather than inventing top-level
fields. `details` is free-form; the top level is a contract.

---

## The four rules

### 1. Emit before every `meta: end_host`

`end_host` stops the play for that host **immediately**. No later task runs —
and `post_tasks` do **not** run afterwards either.

A playbook that emits only at the end reports nothing for any host that exits
early, and the portal cannot tell *"correctly skipped"* from *"never ran"*.
Those mean opposite things to an approver.

`renew_apache_ssl.yml` had exactly this defect on three exit paths, including
the dry-run path the preview flow depends on most.

```yaml
- name: Emit structured result - nothing to do
  ansible.builtin.debug:
    msg: |
      OPSPORTAL_RESULT_BEGIN
      {{ { 'host': inventory_hostname, ... } | to_json }}
      OPSPORTAL_RESULT_END
  when: _nothing_to_do

- name: Skip host
  ansible.builtin.meta: end_host
  when: _nothing_to_do
```

### 2. Read normalised values from `localhost`

`set_fact` is **host-scoped**. A first play that normalises inputs on
`localhost` does not make those values available in a second play running on
the targets — there, `{{ request_id }}` resolves back to the raw survey value.

A trailing space then produces a backup path containing a space, which breaks
`restore.sh` and the portal's rollback — invisibly, until a rollback is
attempted.

```yaml
vars:
  _req_id: "{{ (hostvars['localhost']['request_id'] | default(request_id, true)) | trim }}"
```

### 3. `check_mode: false` on read-only commands

Semaphore's own "Dry Run" checkbox passes `--check` to Ansible, and
`command`/`shell` tasks skip in check mode by default. That is separate from
your `dry_run` survey variable, and the two are easily confused.

Skipped validation is worse than absent validation. In `renew_apache_ssl.yml`
the certificate/key match assertion reported *"All assertions passed"* without
comparing anything, because both operands were undefined and Jinja2 compares
two undefined values as **equal**.

```yaml
- name: Read something
  ansible.builtin.command: "openssl x509 -in {{ f }} -noout"
  changed_when: false
  check_mode: false     # read-only; must run even under --check
```

And never assert on a bare comparison of two registered values:

```yaml
that:
  - _a.stdout is defined
  - _a.stdout | trim | length > 0
  - _a.stdout | trim == _b.stdout | trim
```

### 4. Default `dry_run` to `true`

A dropped or mistyped value must fail safe. The portal sends `dry_run` inside
`params`, but a playbook whose internal default is `false` turns a missing
value into a real production change.

---

## Template for a new playbook

```yaml
  post_tasks:
    - name: Emit structured result for the portal
      ansible.builtin.debug:
        msg: |
          OPSPORTAL_RESULT_BEGIN
          {{
            {
              'host': inventory_hostname,
              'playbook': 'my_automation',
              'ticket_id': _tkt_id,
              'request_id': _req_id,
              'phase': (dry_run == 'true') | ternary('preview', 'execute'),
              'status': _changed | ternary('success', 'already_correct'),
              'changed': _changed,
              'summary': 'Plain sentence a non-technical person can read.',
              'backup_path': _backup_file,
              'rollback_available': true,
              'rollback_command': 'bash ' ~ _backup_file ~ '/restore.sh',
              'details': {
                'whatever': 'this automation needs'
              }
            } | to_json
          }}
          OPSPORTAL_RESULT_END
```

---

## Checking your work

Against a real Semaphore run, from the portal host:

```bash
/opt/ufw-portal/venv/bin/python /opt/ufw-portal/app/fetch_and_parse.py <TASK_ID>
```

`PASS` means the portal will read it. `FAIL` prints why, in order of
likelihood.

Do not test by copying output from the Semaphore web UI — it interleaves a
timestamp line after every output line, and one landing inside a JSON block
breaks the JSON. The portal reads the API, and so does that script.

---

## Where this is heading

Two things depend on this contract, which is why it is worth being strict now.

**The manifest system.** Each playbook gains a `*.manifest.yml` beside it
declaring its plain-language name, its form fields and its approval rules. The
portal discovers them from Git; an admin binds one to a Semaphore template.
Adding automation number three then costs a YAML file and a review — no portal
code. That only works if every playbook reports its outcome the same way.

**Two-way WhatsApp.** Receiving alerts is easy. *Executing* an automation from
a chat message means something must render the form as questions and the result
as a sentence — and it cannot have per-playbook logic baked in. The manifest
supplies the questions; `summary` and `status` supply the answer.

Both reduce to the same requirement: **the playbook describes itself, and
nothing downstream needs to know which playbook it was.**
