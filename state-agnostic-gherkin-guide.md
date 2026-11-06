# State-agnostic, reusable Gherkin steps for ACORD agency quoting

> **How to use this file with GitHub Copilot**
> Save it in the repo (for example `docs/testing/state-agnostic-gherkin-guide.md`, or as a reusable
> prompt under `.github/prompts/`). Then ask Copilot Chat in agent mode:
> *"Read docs/testing/state-agnostic-gherkin-guide.md. Run the Exploration Tasks in section 9 against
> this solution and report findings before changing any code."*
> Work through the phases in section 10 one at a time and review each before moving on.

---

## 1. Context

- The system under test is an agency quoting / rating service. Comparative raters send ACORD XML
  (`PersAutoPolicyQuoteInqRq`) with Progressive extensions (`com.progressive_*`). The service returns
  `PersAutoPolicyQuoteInqRs`, and may also return a second personalized quote
  (`com.progressive_PersAutoPolicyPersonalizedRateQuoteInqRs`).
- Quotes run as a rate-call sequence: `com.progressive_RateCallCd` = RC0, RC1, RC2, ...
- The application is a reusable core plus state-driven variations. Many project teams contribute.
  The goal is **one shared automation suite** that every team extends, not a copy per team or per state.
- Stack: C#/.NET, Reqnroll (BDD), NUnit/MSTest. **ACORD model classes already exist in the
  solution.** Reuse them and do not hand-build XML.

## 2. Problems this design must solve

1. Step definitions or scenarios that hard-code a state (`Given a Texas driver ...`).
2. The same step written several ways by different teams.
3. Large hand-maintained XML files copied per scenario, which drift and contradict themselves.
4. Scenarios that can't explain what they test, because the rule is buried under 100 lines of data.
5. Assertions that only check "premium > 0" and miss what the service actually changed.

## 3. Design principles (non-negotiable)

1. **A scenario describes a rule, not a state.** The state is a value in an `Examples` row.
2. **State-specific values live in data (state profiles), never in step text or step code.**
   No `if (state == "MI")` in step definitions.
3. **Start from a valid baseline and override only what the scenario is about.** The first step
   produces a complete, valid request for the state. Every later step only describes a change.
4. **Business words in Gherkin, ACORD paths in one place.** Gherkin says `yearsLicensed`. A field
   dictionary maps it to the ACORD model property.
5. **One step definition per intent.** Number of drivers, columns and states are data, not new steps.
6. **Assert in business terms, and on changes as well as outcomes.**
7. **Relative assertions over exact premiums.** Exact-premium checks belong to a separate,
   small, owned golden-master set refreshed on each rate change.
8. **Fail loudly on unknown inputs.** An unknown column, field, persona or state fails the test with
   a message naming what's missing. Never ignore it silently.

## 4. Architecture

```
Feature files (rules, tagged by state/LOB)
      │
Step definitions (thin: one line of intent each)
      │
Request composer ──► Base template per LOB (cleaned RC0 request, deserialized into ACORD models)
      │                  + State profile overlay (state defaults)
      │                  + Scenario overrides (via Field Dictionary → model property)
      │                  + Service controls (Debug block: mocks, feature flags)
      │                  + Environment plumbing (signon, producer, env) from config
      ▼
Rating client (interface: real HTTP / WireMock / fake)
      │
Response view (business accessors per quote: standard | personalized; change log; POS messages)
```

### 4.1 The four zones of a request

| Zone | Examples in the request | Owner | Appears in Gherkin? |
|---|---|---|---|
| Risk data | drivers, vehicles, coverages, prior policies, garaging, household | Scenario steps | Yes, only what the rule needs |
| State defaults | garaging address, state minimum coverages, prior-insurance shape | State profile (data) | No, referenced as "the state minimums" |
| Plumbing | `SignonRq`, `Producer`, eSign acknowledgements, `RateCallCd`, `RateCallOrigin`, `PProEnv` | Template + environment config | No |
| Service controls | `Debug/ServiceImplementationOverrides`, `FeatureManager` flags | Dedicated steps | Yes, when the rule depends on them |

### 4.2 Base templates

- One cleaned template per line of business (start with personal auto RC0). Store it as XML or JSON
  and deserialize it into the ACORD models.
- **Clean it first:** remove contradictions such as `YearsLicensed` vs `MonthsLicensed` vs `LicensedDt`,
  or `PurchaseDt` vs `LengthOfOwnership`. Replace real-looking identifiers (SSN, bank account,
  routing number) with references to named test identities.
- All dates in the template are relative to the effective date and resolved at build time.

### 4.3 State profiles (data)

One entry per state. Business owners review it like code.

```json
{
  "TX": {
    "garagingAddress": { "addr1": "...", "city": "...", "postalCode": "..." },
    "requiredCoverages": ["BI", "PD"],
    "minimumLimits": { "BI": "30000/60000", "PD": "25000" },
    "stateMinimumCoverages": { "BI": "30000/60000", "PD": "25000", "UM": "30000/60000", "PIP": "2500" },
    "rejectionNeedsSignedForm": ["UM", "PIP"],
    "fieldsNotAllowed": [],
    "fieldsIgnoredInRating": []
  },
  "CA": {
    "fieldsIgnoredInRating": ["driver.gender"],
    "fieldsRemovedByService": ["policy.ubiAcknowledgement", "policy.ubiEnrollment", "vehicle.ubiEnrollment"]
  }
}
```

*(Values are illustrative. The business confirms the real rules.)*

### 4.4 Field dictionary (data)

Maps business field names to ACORD model properties, with defaults and validation. Adding a field
means adding a row. No step code changes.

| entity | field | model property / ACORD element | default | allowed values | computed from | states |
|---|---|---|---|---|---|---|
| driver | age | `DriverInfo.PersonInfo.BirthDt` | 35 | | effective date − age | all |
| driver | gender | `DriverInfo.PersonInfo.GenderCd` | M | M, F, X | | all (ignored in rating in some) |
| driver | maritalStatus | `DriverInfo.PersonInfo.MaritalStatusCd` | S | S, M, ... | | all |
| driver | relation | `PersDriverInfo.DriverRelationshipToApplicantCd` | IN | IN, SP, CH, ... | | all |
| driver | licenseState | `DriverInfo.License.StateProvCd` | policy state | US states | | all |
| driver | yearsLicensed | `LicensedDt` + `com.progressive_YearsLicensed` + `com.progressive_MonthsLicensed` | 15 | | effective date | all (kept consistent) |
| driver | goodStudent | `PersDriverInfo.GoodStudentCd` | N | Y, N | | all |
| vehicle | vin | `PersVeh.VehIdentificationNumber` | from vehicle profile | | | all |
| vehicle | rideshare | `com.progressive_VehicleUsedForRideshareInd` | 0 | 0, 1 | | all |
| vehicle | annualMiles | `EstimatedAnnualDistance.NumUnits` | 12000 | | | all |
| coverage | BI | `PersVeh.Coverage[CoverageCd=BI].Limit[]` | state minimum | | | all |

Rules:
- One business field can set several ACORD elements, so they stay consistent (for example `yearsLicensed`).
- A computed field (age, years licensed, ownership length) is always relative to the effective date.
- The `states` column lets the suite flag misuse, such as sending an FL-only field in OH.

### 4.5 Service controls

Wrap the `Debug` block in typed steps. Build the `FeatureManager` value from a dictionary so keys
can't be duplicated (the current sample repeats flags).

## 5. Step vocabulary (target)

### 5.1 Start (one per line of business)

```gherkin
Given a personal auto quote request for "<state>"
Given a home quote request for "<state>"
```

### 5.2 Shape the risk

```gherkin
# Wide table: optional columns, blank cell = default, table REPLACES default drivers
And the request has these drivers:
  | relation | age | maritalStatus | yearsLicensed | goodStudent |
  | IN       | 45  | M             | 25            |             |
  | SP       | 43  | M             | 22            |             |
  | CH       | 17  | S             | 1             | Y           |
  | CH       | 19  | S             | 3             |             |

# Vertical table: patch ONE driver with many fields
And driver 3 has:
  | field          | value |
  | distantStudent | Y     |
  | driverTraining | Y     |

# Repeating children: separate table keyed to the entity
And driver 4 has these incidents:
  | type            | date        | atFault |
  | AtFaultAccident | {today-10m} | Y       |

# Named profiles for common shapes
And the request has a "newly-licensed-teen" driver
And the household is "large-household-12-drivers"
And the request has 6 drivers based on "standard-adult"

# Same patterns for vehicles, coverages, prior insurance
And the request has these vehicles:
  | vin               | use | annualMiles | primaryDriver |
  | {vin:2021-RAV4}   | PL  | 9000        | 1             |
And the request has these coverages:
  | coverage | limit         | deductible |
  | BI       | 100000/300000 |            |
  | COLL     |               | 500        |
And coverage "UM" is removed
And coverage "UM" is rejected with a signed rejection form
And the prior insurance is "continuous-3-years"

# Identities by name: never put PII in feature files
And the insured is the "credit-demo-b1" identity
```

### 5.3 Service controls and rate calls

```gherkin
And the "VehicleHistory" service returns "NotOrdered"
And feature flag "avq_choice_rate_flag" is "Y"
When the rater submits the RC0 quote request
When the rater submits the RC1 quote request          # reuses carrier IDs and quote number from RC0
```

### 5.4 Escape hatch (use sparingly)

```gherkin
And the request field "PersPolicy/ContractTerm/EffectiveDt" is "{today+90d}"
And the request field "PersPolicy/ControllingStateProvCd" is removed
```

If the same path shows up in three or more scenarios, promote it to a dictionary field.

### 5.5 Assertions

```gherkin
Then the "standard" quote succeeds                    # Success or SuccessWithChanges
Then the "personalized" quote is offered
Then the quote is rejected
And the rejection reason mentions "<coverage>"
And every coverage required by the state is in the "standard" quote
And the "personalized" quote premium is higher than the "standard" quote premium
And the second premium is lower than the first

# Change log (ChangeStatus A/M/D resolved to business fields)
Then the service changed vehicle 1 "antiTheftDevice" from "1" to "N"
And the service added vehicle 1 "make" and "model" from the VIN
And the service removed the fields listed for the state      # from state profile
And the service made no other changes to driver 1

# Point-of-sale messages
And the point-of-sale messages include "Multi Policy Discount"

# Generic fallback
And the response field "MsgStatus/MsgStatusCd" is "SuccessWithChanges"
```

### 5.6 Tokens

`{today}`, `{today+30d}`, `{today-14m}`, `{today-16y}`, `{effective+6m}`, `{guid}`, `{blank}`,
`{vin:<profile>}`. Resolved at build time so the data never goes stale.

## 6. Scenario patterns

```gherkin
@lob_AUTOP @layer_api
Feature: Personal auto rating follows each state's rules

  @smoke
  Scenario Outline: A standard risk gets a premium in every supported state
    Given a personal auto quote request for "<state>"
    When the rater submits the RC0 quote request
    Then the "standard" quote succeeds
    And every coverage required by the state is in the "standard" quote

    @state_TX
    Examples:
      | state |
      | TX    |
    @state_CA
    Examples:
      | state |
      | CA    |

  @regression
  Scenario Outline: A coverage the state requires cannot be removed
    Given a personal auto quote request for "<state>"
    And coverage "<coverage>" is removed
    When the rater submits the RC0 quote request
    Then the quote is rejected
    And the rejection reason mentions "<coverage>"

    @state_TX
    Examples:
      | state | coverage |
      | TX    | BI       |
    @state_MI
    Examples:
      | state | coverage |
      | MI    | PIP      |

  # A rule only one state has gets its own scenario with a state tag
  @regression @state_CA
  Scenario: Usage-based insurance fields are removed in California
    Given a personal auto quote request for "CA"
    And the request has these vehicles:
      | ubiEnrollment |
      | 1             |
    When the rater submits the RC0 quote request
    Then the "standard" quote succeeds
    And the service removed the fields listed for the state

  # Rating-factor boundary, assessed relatively
  @regression
  Scenario Outline: Gender does not change the premium where it is not a rating factor
    Given a personal auto quote request for "<state>"
    And driver 1 has:
      | field  | value |
      | gender | M     |
    When the rater submits the RC0 quote request
    And the same request is re-rated with driver 1 "gender" set to "F"
    Then the two "standard" premiums are equal

    @state_CA
    Examples:
      | state |
      | CA    |
```

### Choosing the right shape for a lot of data

| Need | Shape |
|---|---|
| A rule about a count (max drivers) | `the request has N drivers based on "<profile>"` + one override |
| Many configurations, each tested alone | One driver profile per `Examples` row |
| One request with many distinct entities | Named household data file + small overrides |
| Combinations of attributes | Seeded pairwise generator from field dictionary allowed values |

A reader should grasp what a scenario proves in about 10 seconds.

## 7. Tags

```
@lob_AUTOP @lob_HOME          line of business
@state_TX ...                 put on Examples blocks so rows are filterable
@smoke @regression            depth
@layer_api @layer_ui          rating service vs agent portal
@ratecall_RC0 @ratecall_RC2   which call the scenario targets
@team_<name>                  ownership (optional)
```

`dotnet test --filter "Category=state_TX&Category=smoke"`

## 8. Anti-patterns to remove

- Step text containing a state name, or step code branching on state.
- Several step definitions with different wording for the same intent.
- Copying a full XML request per scenario to change one field.
- XPath or element names in step definitions.
- Hard-coded dates, ages as birth dates, fixed premiums in functional scenarios.
- PII-like values (SSN, bank account, license number) in feature files, templates or reports.
- Assertions that check only "success" or "premium > 0".
- Gherkin tables wider than about 7 columns or longer than about 6 rows (use vertical tables, profiles or data files).

## 9. Exploration tasks for Copilot (report before changing code)

Run these against the existing solution. Answer each with file paths and short evidence.

1. **Inventory step definitions.** List every `[Given]/[When]/[Then]` with its regex or expression and file.
   Group them by intent and flag duplicates (different wording, same intent).
2. **Find state coupling.** Find step text, step code, helpers or test data that hard-code a
   state code, state name, or branch on state. List each with a suggested data-driven replacement.
3. **Find the ACORD models.** Identify the namespaces and classes for `ACORD`, `PersAutoPolicyQuoteInqRq`,
   `PersDriver`, `PersVeh`, `Coverage`, `PersPolicy`, `Debug`, and how `com.progressive_*` extensions are
   modelled. Check for existing serializer settings (element order, namespaces, empty-element handling).
4. **Find how requests are built today.** XML files on disk, string templates, builders, or models?
   How many request XML files exist, and how much do they differ? Identify a candidate base template.
5. **Map existing data to the field dictionary.** For each field used by current scenarios, propose
   a dictionary row (entity, business name, model property, default, allowed values).
6. **Find response assertions.** How are responses read today? Are the standard and personalized
   quotes told apart? Is `ChangeStatus` used? Are `PosMsg` messages checked?
7. **Find service control usage.** Where is the `Debug` block set? Which overrides and feature flags
   are used, and are they duplicated?
8. **Find PII in test assets.** List files containing SSN-like, bank account, routing or license
   number patterns, with a proposal to move them into named identities.
9. **Assess fit.** For each section of this guide, say: already supported / partially / missing,
   and estimate the effort to close the gap.

## 10. Phased refactoring plan (one PR per phase)

1. **Field dictionary and state profiles** as data files with loaders and validation (unknown key → fail).
2. **Request composer**: base template → state overlay → overrides → service controls → plumbing,
   using the ACORD models. Unit-test the composer without calling the service.
3. **Response view**: per-quote accessors, change-log resolution (`IdRef` → element → business field),
   POS messages, premiums by policy, vehicle and coverage.
4. **Generic step definitions** from section 5, keeping the old steps as thin wrappers marked `[Obsolete]`
   during migration.
5. **Migrate one feature file per team** to the new vocabulary, then remove the obsolete steps.
6. **Rate-call sequence** (RC0 → RC1 → RC2): carry carrier IDs and quote numbers in scenario context.
7. **Pairwise generator** for combinatorial driver and vehicle coverage, seeded and reproducible.

## 11. Definition of done

- Adding a new state needs only data (state profile + Examples rows), unless the state has a unique rule.
- Adding a new driver or vehicle field needs only a dictionary row.
- No step definition contains a state code, XPath or element name.
- Every failure message shows the response summary (status, quote, premium, messages, changes), and
  the request and response XML are attached to the report with PII masked.
- Two teams can add scenarios for different states in the same feature file without touching step code.
