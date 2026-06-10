# ACES 4.2 Fitment Attributes & Rules (Summary)

> **Scope**: Fitment (Application) content only. Digital Asset/diagram content is intentionally excluded.

---

## 1. File Structure (Fitment-Focused)

**Root element**
- The file must have a single `<ACES>` root element with a `version` attribute. Valid values: `4.0`, `4.1`, `4.2`. The root wraps Header, App, and Footer sections.

**Physical format**
- The transfer file is plain-text XML using UTF-8 encoding, with CR/LF line endings.

**Naming convention**
- `<Company>_<CatalogTitle>_<YYYY-MM-DD>_<FULL|UPDATE|TEST>.xml` (zip file naming rules also apply when compressing multiple files).

---

## 2. Header & Version Tags

**Required header rules**
- Header tags are required, ordered per the XML schema.

**Version header tags**
- `PcdbVersionDate` identifies the PCdb version used for Part Terminology (PartType) and Position coding.
- `QdbVersionDate` identifies the Qdb version used for Qualifier coding.

**Optional header tags introduced/extended in 4.2**
- `PartsApprovedFor`, `RegionFor`, and `SubBrandAAIAID` are listed as optional header-level elements in the 4.2 revision notes.

---

## 3. Application (Fitment) Element Overview

**Application record (`<App>`)**
- Each `<App>` groups information that defines a fitment. `action` and `id` are required. `action` is `A` (add) or `D` (delete). `id` is a unique sequential ID in the file. `ref` is optional. `validate` defaults to `yes` and, if set to `no`, the receiver will skip VehicleTo validation.

**Required application content**
- Each application must include: **Vehicle Identification**, **Part Type**, **Part Number**, and **Per Car Quantity**.
- Optional content includes **Vehicle Attributes**, **Comments**, and **Position**.

---

## 4. Vehicle Identification (Fitment Scope)

**Core rule**
- Each application must contain either a **Base Vehicle** (includes year) **or** a **Make + Year-Range** combination.

**Equipment & system applications**
- ACES supports equipment applications (equipment base or manufacturer + model + vehicle type). ACES 4.2 adds cataloging by **vehicle system** using the equipment format.
- System attributes not in equipment format can be expressed using Qdb qualifiers.

---

## 5. Part Type (Part Terminology)

- Each application must include a **PCdb Part Terminology ID** (lowest level). Position info, if supplied, is validated against this Part Terminology ID.

---

## 6. Part Number

- Provide a single part number formatted for end-user display. **Do not embed** quantity, position, or footnote info in the part number.

---

## 7. Per Car Quantity (Required)

- Every application must include a **Per Car Quantity**.
- If position-specific (e.g., Left Rear), the quantity should reflect that position (often `1`). If a part applies to multiple positions without full position detail, the quantity should reflect the **total** (e.g., `2` for Left + Right Rear).
- Single-item parts (e.g., lug nuts) should generally use quantity `1`. For packaged sets (e.g., gasket set), quantity should be `1`.

---

## 8. Vehicle Attributes (Optional)

- Vehicle attribute tags are **VCdb ID fields** that further narrow the application scope. Multiple attributes are logically combined with **AND**.
- You **cannot** use two of the same attribute tag in a single application (e.g., Coupe and Sedan).
- Do **not** use vehicle attribute IDs that are “abbreviation” placeholders (e.g., `N/A`, `U/K`, `N/R`, or `-`).

**Vehicle attribute tag names (vehicle-only, no equipment tags)**
- Vehicle identification: `BaseVehicle`, `Make`, `Model`, `SubModel`, `Region`
- Body/bed: `BodyType`, `BodyNumDoors`, `BedType`, `BedLength`, `WheelBase`, `MfrBodyCode`
- Drive/aspiration: `DriveType`, `Aspiration`
- Engine: `EngineBase`, `EngineBlock`, `EngineBoreStroke`, `EngineDesignation`, `EngineMfr`, `EngineVersion`, `EngineVIN`, `CylinderHeadType`, `ValvesPerEngine`
- Fuel/ignition: `FuelDeliveryType`, `FuelDeliverySubType`, `FuelSystemControlType`, `FuelSystemDesign`, `FuelType`, `IgnitionSystemType`
- Brakes/suspension: `BrakeABS`, `BrakeSystem`, `FrontBrakeType`, `RearBrakeType`, `FrontSpringType`, `RearSpringType`
- Steering: `SteeringType`, `SteeringSystem`
- Transmission: `TransmissionBase`, `TransmissionControlType`, `TransElecControlled`

All vehicle attribute tags use an `id` attribute referencing the corresponding VCdb table.

**Except logic**
- Except logic is not directly supported with vehicle attributes. Translate to positive logic.
- For vehicles with unknown attributes (e.g., N/A, U/K), put the excepted expression in a **Note**.

---

## 9. Qualifiers (Qdb)

**Basic qualifier structure**
- Use `<Qual>` elements to reference Qdb qualifiers with a required `text` element. Qualifiers must be **coded** (referencing Qdb IDs).
- The `text` element is required to support early adoption; **Note** should be used only for excepted attribute strings when attributes are unknown.

**Qualifier parameters**
- Use `<param>` elements for qualifiers that require parameter values. Parameter order **must** match the numbered placeholders in the qualifier expression.
- Decimal or fractional values are allowed (e.g., `3/4`, `1-1/2`). Mixed fractions use `#-#/#`.
- Some parameter types require `uom` (unit of measure). Valid uom values include items like `in`, `mm`, `lb`, `kg`, `g`.
- Alternate units can be specified with `altvalue` and `altuom`.

**Complex OR logic**
- Since application attributes are AND-ed, **OR logic** should be represented by creating **multiple applications** (one per branch).

---

## 10. Change Delivery Rules

- A full file uses `action="A"` for all applications. Updates are delivered as paired `D` (delete old) + `A` (add new). Year-range changes can be made by deleting the specific year and adding the updated year.

---

## 11. General Rules & Invalid Applications

- Applications should **stand alone** and include only what is necessary to select the correct part.
- Applications with invalid vehicle/attribute combinations are discouraged. Only un-researched attributes (`N/A`, `U/K`) are acceptable in normal coded tags.

**Special “Not Required / Not Available / Not Serviceable” guidance**
- `N/R`: If a part is not required on a vehicle, **omit** the application rather than encoding a negative fitment.
- `N/A` (Not Available): Do **not** use as a placeholder for future applications. Wait until valid info exists.
- `N/S` (Non‑Serviceable): Use only when a part exists but is non‑serviceable; if a solution exists, ensure the fitment note specifies modification to original vehicle configuration.

---

## 12. Full Element List

- The full XML element catalog is defined in **Section 6.12 XML Elements** of the ACES 4.2 documentation. Use that list for exhaustive tag coverage.
