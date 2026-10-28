# Lab 05 — S3 Security: Private by Default, Guardrails, Least Privilege

**Date:** 28 October 2026
**Region:** Asia Pacific (Sydney) — ap-southeast-2
**Setup:** Windows 11 · PowerShell · AWS CLI v2 · IAM user `talal-admin`
**Status:** Complete — proofs captured, bucket, versions, key, user,
policy and local profile all deleted

> All account IDs, user IDs, policy IDs, owner IDs and organisation IDs
> in this write-up are masked with `************`. No credential appears
> anywhere in this repository. There is deliberately no screenshot of
> the access-key creation step, and no screenshot in this write-up at
> all — the terminal captures from this sitting contain account and
> owner IDs in ARN and Owner lines, so the verified outputs are quoted
> in text, with those identifiers redacted, instead of publishing the
> raw captures.

## Objective

Prove, with live calls rather than console settings, that an S3 bucket
starts private, that Block Public Access actually blocks a public
policy, that encryption and versioning protect the data, and that a
least-privilege reader can read exactly one bucket and nothing else —
then remove every trace of the lab in dependency order.

Cost for the whole lab: **$0** — only tiny text files were stored, and
only for the duration of the sitting.

## Step 1 — Bucket created private by default

```powershell
aws s3api create-bucket --bucket talal-lab05-s3-20261028 --region ap-southeast-2 --create-bucket-configuration LocationConstraint=ap-southeast-2
aws s3api get-public-access-block --bucket talal-lab05-s3-20261028
```

Verified on 28 October 2026:

- Bucket created: `talal-lab05-s3-20261028`
- All four Block Public Access settings **true**: `BlockPublicAcls`,
  `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`

A new bucket is private before anything is configured on it — the
guardrail is already on, not something added afterwards.

## Step 2 — Private, proven by an unauthenticated 403

```powershell
Set-Content -Encoding ascii test.txt "Lab 05 test file - private by default"
aws s3 cp test.txt s3://talal-lab05-s3-20261028/
curl.exe https://talal-lab05-s3-20261028.s3.ap-southeast-2.amazonaws.com/test.txt
```

→ Unauthenticated request returned **`AccessDenied`** (403). The file
uploads fine as the owner and is invisible to everyone else. That 403
is the proof — if the file had opened, the bucket would have been
public, which is exactly the failure this lab is about.

## Step 3 — The guardrail blocks a public policy

A public-read policy was drafted (Notepad, ANSI encoding — the Lab 04
BOM lesson applied from the start) granting `s3:GetObject` to
`Principal: "*"` on the bucket's objects, then applied:

```powershell
aws s3api put-bucket-policy --bucket talal-lab05-s3-20261028 --policy file://public-read-policy.json
```

→ **Blocked.** AWS returned `AccessDenied` stating that public
policies are prevented by the `BlockPublicPolicy` setting.

Two points worth recording:

1. The block was tested **without turning Block Public Access off**.
   Turning it off "just to see if the policy works" would have been
   the exposure itself — the lab teaches the guardrail by watching it
   hold, not by disabling it.
2. Because the policy was never applied, there was nothing to remove
   at cleanup — `delete-bucket-policy` later ran without error.

## Step 4 — Encryption, versioning, and two real versions

### Encryption: two failures before the fix that worked

The first attempt failed — PowerShell stripped the double quotes from
an inline JSON argument, so AWS received `{Rules:...}` with no quotes
and rejected it as invalid JSON. This is a known PowerShell quoting
trap, not a JSON error.

The second attempt failed differently: the JSON was moved to a file,
but the file was empty (0 bytes — Notepad had created it without
saving the pasted content), so AWS returned a parameter-parsing error
about an empty input. The diagnosis was to check the file before
re-running anything:

```powershell
Get-Content bucket-encryption.json
dir bucket-encryption*
```

→ Length 0 confirmed the empty file. The fix was to reopen the file,
paste the JSON, save with Ctrl+S, and verify with `Get-Content`
before re-running:

```powershell
aws s3api put-bucket-encryption --bucket talal-lab05-s3-20261028 --server-side-encryption-configuration file://bucket-encryption.json
aws s3api get-bucket-encryption --bucket talal-lab05-s3-20261028
```

→ Verified: `SSEAlgorithm: AES256` (SSE-S3 default encryption).

Lesson to keep: *when a file-based AWS argument fails to parse, check
the file's contents and length first — an empty file and a quoting
problem look identical in the error, and re-running the same command
fixes neither.*

### Versioning: v1 and v2, identified by evidence, not labels

```powershell
aws s3api put-bucket-versioning --bucket talal-lab05-s3-20261028 --versioning-configuration Status=Enabled
Set-Content -Encoding ascii test.txt "Lab 05 test file - version 2"
aws s3 cp test.txt s3://talal-lab05-s3-20261028/
aws s3api list-object-versions --bucket talal-lab05-s3-20261028
```

S3 does not label versions "v1" and "v2" — they are identified by
`IsLatest`, size and timestamp. The list showed both:

- **v2:** `IsLatest: true`, Size 30, last modified 2026-10-28
  10:47:36 — the overwritten upload
- **v1:** `VersionId: null`, `IsLatest: false`, Size 39, last
  modified 2026-10-28 10:28:24 — the original upload, made before
  versioning was enabled, preserved as the null version

Overwriting did not destroy the old file. That is the whole point of
versioning: an overwrite (or a delete) is recoverable, not final.

## Step 5 — Least-privilege reader, proven by one PASS and two denies

Policy intent: `s3:ListBucket` on this bucket only, and
`s3:GetObject` on this bucket's objects only — nothing else, no other
bucket.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::talal-lab05-s3-20261028" },
    { "Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::talal-lab05-s3-20261028/*" }
  ]
}
```

```powershell
aws iam create-policy --policy-name Lab05S3Reader --policy-document file://lab05-reader-policy.json
aws iam create-user --user-name lab05-reader
aws iam attach-user-policy --user-name lab05-reader --policy-arn <policy-Arn-from-your-own-screen-do-not-paste-it-in-chat>
aws iam create-access-key --user-name lab05-reader
aws configure --profile lab05-reader
# Access Key ID -> from the create-access-key output
# Secret Access Key -> from the same output, typed, never pasted in chat
# Region -> ap-southeast-2
# Output -> json
```

**No screenshot of the create-access-key terminal, and no secret in
chat** — the Lab 02 rotation lesson applies again. Two environment
notes from this step: the newer AWS CLI also asks whether to
configure AI-agent skills — answered `never`, it is unrelated to the
lab and stops the prompt recurring — and `aws configure --profile`
writes **two** files, `credentials` and `config`, which matters at
cleanup (Lab 04, same note).

**The proof itself — one PASS, two live denies:**

```powershell
aws s3 cp s3://talal-lab05-s3-20261028/test.txt reader-download.txt --profile lab05-reader
```

→ **PASS.** The reader downloaded the file.

```powershell
aws s3 cp test.txt s3://talal-lab05-s3-20261028/upload-test.txt --profile lab05-reader
aws s3 ls --profile lab05-reader
```

→ Both **`AccessDenied`** — no identity-based policy allows
`s3:PutObject` on this bucket, and no policy allows
`s3:ListAllMyBuckets`. Those two failures **are** the evidence that
the policy boundary holds — do not "fix" them. A reader that can
read one bucket, write to none, and cannot even list the account's
other buckets is exactly what least privilege looks like in practice.

## Step 6 — Cleanup, in dependency order

Delete in the wrong order and AWS refuses — so versions first, then
the bucket, then the identity, then the local profile:

```powershell
aws s3api list-object-versions --bucket talal-lab05-s3-20261028
aws s3api delete-object --bucket talal-lab05-s3-20261028 --key test.txt --version-id <full-v2-version-id-from-the-list>
aws s3api delete-object --bucket talal-lab05-s3-20261028 --key test.txt --version-id null
aws s3api delete-bucket-policy --bucket talal-lab05-s3-20261028
aws s3api delete-bucket --bucket talal-lab05-s3-20261028
```

One stumble, worth recording: the first `detach-user-policy` attempt
failed with `InvalidInput` because the **user** ARN (ending
`user/lab05-reader`) was pasted instead of the **policy** ARN (ending
`policy/Lab05S3Reader`). The policy ARN was recovered without
re-creating anything:

```powershell
aws iam list-policies --scope Local --query "Policies[?PolicyName=='Lab05S3Reader'].Arn"
```

Then, with the correct ARN:

```powershell
aws iam list-access-keys --user-name lab05-reader
aws iam delete-access-key --user-name lab05-reader --access-key-id <key-id-from-the-create-output>
aws iam detach-user-policy --user-name lab05-reader --policy-arn <policy-Arn>
aws iam delete-user --user-name lab05-reader
aws iam delete-policy --policy-arn <policy-Arn>
```

Then the local profile — from **both** files: delete the
`[lab05-reader]` section from `credentials` and the
`[profile lab05-reader]` section from `config`, leaving `[default]`
(`talal-admin`) untouched. Verified afterwards:

```powershell
aws s3api head-bucket --bucket talal-lab05-s3-20261028
aws iam get-user --user-name lab05-reader
```

→ HeadBucket = **404 Not Found**. GetUser = **NoSuchEntity**.
Nothing lab-related remains: no bucket, no versions, no key, no user,
no policy, no local profile. The `[default]` profile was verified
untouched.

## Security takeaways

- **Private by default is a starting state, not a setting you add.**
  A new bucket already had all four Block Public Access settings on —
  the lab's job was to prove it, not to configure it.
- **A guardrail is proven by watching it block, not by disabling it.**
  The public policy was never applied, so the cleanup had nothing to
  undo — that is the correct outcome, not an incomplete lab.
- **Quoting and file contents are failure modes, not typos.**
  PowerShell stripping JSON quotes and a 0-byte Notepad file produced
  two different-looking errors with the same root cause class: what
  AWS actually received was not what was on screen. Verify the input
  (`Get-Content`, file length) before re-running the command.
- **Versions are identified by evidence, not labels.** S3 never
  writes "v1" or "v2" — `IsLatest`, size and timestamp are the
  identifiers, and the pre-versioning object survives as the null
  version.
- **A policy is a claim; an AccessDenied is evidence.** The reader's
  boundary is only proven when download passes and upload plus
  bucket-listing both fail.
- **ARNs are not interchangeable.** A user ARN and a policy ARN look
  alike at a glance; pasting the wrong one into `detach-user-policy`
  fails with `InvalidInput`, and the fix is to look the policy ARN up
  rather than guess it.
- **Cleanup is two files, not one.** `aws configure --profile` writes
  `credentials` and `config`; removing the profile means both, or
  the profile lingers.
- **Never publish a terminal capture unexamined.** ARN and Owner
  lines in this sitting's captures carry account and owner IDs — that
  is why this write-up quotes outputs in text instead of embedding
  the raw screenshots.

*Account IDs, user IDs, policy IDs, owner IDs and organisation IDs are
masked or redacted in this write-up. No credential appears anywhere in
this repository. Practice resources are removed at lab end; the
`[default]` profile is left exactly as found.*
