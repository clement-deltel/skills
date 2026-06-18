---
name: env-config-generator
description: "Generate a structured, production-ready environment-variables configuration file for any service. Use this skill whenever the user asks to create or generate a .env file from service documentation, produce an environment-based configuration file, convert official docs into a .env, improve or expand an existing .env using official documentation, or configure a self-hosted service via environment variables. Triggers include: 'generate a config file for X', 'create a .env for X', 'build an env config from these docs', 'configure X with environment variables', or when the user provides a link to service documentation alongside a request to produce a config file."
compatibility: web_fetch (internet access required for URL inputs), bash_tool or view (for uploaded file inputs)
---

# Environment Variables Configuration File Generator

Generate well-structured, documented `.env` configuration files by synthesizing service documentation, reference configs, and/or existing base files into a single, commented, section-organized output.

## Input

One or more of the following (additive — combine all provided inputs):

- **Documentation URL(s)**: Official configuration reference, cheat sheet, or environment-variables page
- **Reference config**: Official sample config from the service repo (e.g., `.env.example`)
- **Base config**: An existing `.env` file to improve or extend

## Instructions

### Step 1 — Fetch and read all inputs

For every URL provided, call `web_fetch` to retrieve the full page. For uploaded files, use `view` or `bash_tool` to read them. Do this for all inputs before writing anything.

Extract from the documentation:
- The **service name** (used to derive the env-var prefix, e.g., `SERVICE`, `POSTGRES`)
- The **env-var naming convention** (e.g., `SERVICE__section__KEY`, `SERVICE_KEY`, or just `KEY`)
- The **full list of sections and parameters**, with descriptions, default values, and allowed values
- Any **translation mechanism** (e.g., Service's `environment-to-ini`, Docker `_FILE` suffixes)

If only a config file is provided with no URL, infer the format from the file itself and note that descriptions were inferred rather than sourced from official docs.

### Step 2 — Map sections and parameters

Organize all parameters into a hierarchy that matches the documentation's section structure. For each parameter, capture:

- Full env-var key (applying the naming convention)
- Default value
- One-sentence description extracted from the docs
- Allowed values if enumerated (e.g., `[local, minio]`, `[smtp, dummy]`)
- Whether the parameter is sensitive (secret)

Subsections become `Parent - Child` labels in the output (e.g., `Repository - Pull Request`).

### Step 3 — Identify secrets

A parameter is a secret if it is any of: password, token, secret key, JWT secret, API key, signing key, internal token, private key, encryption key, or credential of any kind.

For secrets, replace the value with `${PREFIX_DESCRIPTIVE_NAME}` where:
- `PREFIX` = service name in UPPERCASE (e.g., `SERVICE`, `SMTP`)
- `DESCRIPTIVE_NAME` = concise identifier (e.g., `SECRET_KEY`, `DB_PASSWORD`, `JWT_SECRET`, `LFS_JWT_SECRET`)

If the docs mention a generation command, include it on the line above as a comment (e.g., `# Generate with: pwgen --capitalize --numerals --secure 512 1`).

Never hardcode secret values. Never leave a secret as a bare string.

### Step 4 — Write the output file

Assemble the output following the exact formatting rules below.

**File header** (always first):
```
# ---------------------------------------------------------------------------- #
#               ------- <Service Name> Configuration File ------
# ---------------------------------------------------------------------------- #
# <documentation URL if provided>
```

**Translation note** (if the service uses an env-to-config translation mechanism):
```
# <one or two lines explaining how env vars map to the config format>
# e.g. Format: SERVICE__<section>__<KEY>=value
```

**Section headers** (one per logical section or subsection):
```
# ---------------------------------------------------------------------------- #
#               ------- <Section Name> ------
# ---------------------------------------------------------------------------- #
```

**Parameters** (one per line, comment immediately above):
```
# Brief one-sentence description (include allowed values if enumerated).
PARAM_KEY=default_value
```

**Commented-out parameters** (for optional settings with no default or not commonly needed):
```
# Description.
# PARAM_KEY=
```

**Multi-line descriptions** are allowed when the parameter warrants it (security warnings, generation instructions, architectural notes):
```
# Secret used to validate internal communication.
# Generate with: openssl rand -base64 64
SERVICE__security__INTERNAL_TOKEN=${SERVICE_INTERNAL_TOKEN}
```

### Step 5 — Flag ambiguities (optional)

If any parameters are genuinely ambiguous (conflicting documentation, unclear defaults, deployment-specific values that cannot be inferred), append a `Manual review required` section at the very end:

```
# ---------------------------------------------------------------------------- #
#               ------- Manual Review Required ------
# ---------------------------------------------------------------------------- #
# The following parameters require site-specific decisions:
#
# SOME__parameter__KEY
#   Reason: <explain the ambiguity or conflict>
#
# ANOTHER__parameter__KEY
#   Reason: <explain why this cannot be auto-determined>
```

Limit this section to genuine blocking ambiguities. Do not flag parameters that simply have no default — they should be commented out instead.

## Output

A single `.env` file.

The file should:
- Cover all sections from the documentation (not just common ones)
- Be ordered to match the documentation's section order
- Use the service's actual env-var naming convention throughout
- Contain a comment for every uncommented parameter
- Use `${PLACEHOLDER}` references for all secrets (never bare values)
- Be immediately usable as a starting point with only secrets and host-specific values left to fill in

## Edge Cases

**Multiple documentation URLs**: Fetch all of them. Merge parameters from all sources; documentation URLs take precedence over reference config files for descriptions; the base config takes precedence for values.

**Base config provided without documentation**: Improve it by adding missing comments and standardizing formatting, but note at the top that descriptions may be incomplete without a documentation source.

**Service with flat env vars** (no section nesting, e.g., `DB_HOST=`, `APP_PORT=`): Use the same header and section-marker structure, but emit parameters with their flat names. Sections can group related flat vars (e.g., `Database`, `Server`).

**Very large documentation pages**: Focus on the most operationally relevant sections first (database, server, security, auth). Note at the top if sections were omitted for brevity, and which ones.

**Conflicting defaults** between documentation and reference config: Prefer the reference config value (it's closer to what ships), note the conflict in a comment above the line.

**Docker Compose / secrets files** (`_FILE` suffix convention): If the service supports this pattern, note it in a comment near relevant secrets (e.g., `# Also supports PARAM_FILE=/run/secrets/...`).
