# test5

Here’s a detailed comparison of the **Monex** and **Nium** corporate onboarding APIs, including their request and response structures, and a clear summary of what you’d need to handle to abstract both behind a unified codebase.

***

## **Request Comparison**

| Field/Aspect                  | Monex `/api/entities/create`                                                                                                                                          | Nium `/api/v1/client/{clientId}/applications`                                                                |
|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| **HTTP Method**               | POST                                                                                                                                                                 | POST                                                                                                         |
| **Base Path**                 | `/api/entities/create`                                                                                                                                               | `/api/v1/client/{clientId}/applications`                                                                     |
| **Top-level type**            | `type: "corporate"`                                                                                                                                                  | `applicationType: "corporate"`                                                                               |
| **Company Name**              | `name`, `legalName`                                                                                                                                                  | `corporate.businessName`                                                                                     |
| **Company Registration**      | `identificationNumber` (with possibly `referenceId`)                                                                                                                 | `corporate.businessRegistrationNumber`                                                                       |
| **Address**                   | `city`, `state`, `postCode`, `countryId`, `streetAddress`                                                                                                            | `region` (top-level), `corporate.registeredCountry`                                                          |
| **Contact Phone**             | `companyPhone` (top level), `authorizedContacts[].phone`                                                                                                             | `corporate.applicantEmail` (email only; no phone by default)                                                 |
| **Directors**                 | `corporateDirectors[]` — multiple fields incl. DOB, first/last, ownership, occupation                                                                                | Not in basic example, but may have extended KYC endpoints                                                    |
| **Authorized Contacts**        | Array with name, email, phone, title                                                                                                                                | Not in basic example; basic email only in `corporate`                                                        |
| **Reference Field**           | `referenceId`                                                                                                                                                        | —                                                                                                            |
| **Document Upload**           | Not in this payload, but could exist elsewhere                                                                                                                       | Not in this payload, but may exist in further Nium onboarding steps                                          |

### **Sample Structure Differences**

- **Monex** has richer identity: accepts directors and authorized contacts directly, more granular address, phone, and extra company fields.
- **Nium** wraps company info in a `corporate` object under the top level, has less granular address, and expects fewer nested lists on initial onboarding.

***

## **Response Comparison**

| Field/Aspect                     | Monex Response                                  | Nium Response                                                                                      |
|----------------------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------------|
| **ID Returned**                  | `data.entityId`, `data.entityNumber`            | `applicationId`                                                                                    |
| **Onboarding Form URL**          | No (ID only; further steps may return a URL)    | `url` (link to specific onboarding/customer-facing form)                                           |
| **Status/Expiry Returned**       | No                                              | `expiry` (timestamp, form validity); Nium also often provides status in other flows                |
| **Payload Structure**            | Top-level `data` object                         | Flat JSON (no `` wrapper)                                                                     |

***

## **Delta/Abstraction Requirements for Unified API Call**

- **Field Mapping:**
  - Must map `legalName`/`name` (`Monex`) to `businessName` (`Nium`).
  - Registration number (`identificationNumber` for Monex, `businessRegistrationNumber` for Nium) needs matching.
  - Address must map between multiple scalar fields (Monex) and minimal/region (Nium).
- **Contact Information:**
  - Monex expects phone numbers and list of contacts/directors on Day 1; Nium only expects email, name.
- **Nested Structures:**
  - Your client/customer data model must unify or allow for optional inclusion of lists like directors and authorizedContacts (Monex only).
- **Response Harmonization:**
  - Nium gives an onboarding URL — you may need logic to optionally show or hide this on a per-provider basis.
  - Both give a unique onboarding/application ID; these can be modeled as a common `onboardingId`.
  - If consuming directly, you may want to always `unwrap` Monex’s `data` object for parity.

***

## **If You Use a Single API Call**

- **Minimal Subset:**  
  If you only provide what both APIs require (company name, registration, email, country), onboarding will succeed — but you’ll lose Monex-specific power (directors, contacts, granular address).

- **Full Monex Data:**  
  If your unified onboarding form collects *all* Monex fields, you can always map a subset to Nium and ignore the extras. The reverse is not true.

- **Recommended Delta Logic:**  
  - Ingest superset fields (for both Monex and Nium).
  - When calling Nium, map only the fields it expects.
  - When calling Monex, pass all fields as received.
  - When handling responses, treat both success cases as returning a unique onboarding/application/entity ID with optional external onboarding URL.

***

## **Summary Table**

| Integration Point       | Common Field                     | Nium (Required/Optional)           | Monex (Required/Optional)           |
|------------------------|----------------------------------|-----------------------------------|-------------------------------------|
| Company Name           | businessName / name / legalName  | Required (corporate.businessName) | `name` (required), `legalName` (required) |
| Registration Number    | businessRegistrationNumber/idNum | Required (corporate.businessRegistrationNumber) | `identificationNumber` (required) |
| Address                | address/region/country           | Required (region/corporate.registeredCountry) | Multiple fields (city, state, etc., all required) |
| Contact Email          | applicantEmail                   | Required                          | authorizedContacts[n].email (required but as a list) |
| Contact Phone          | companyPhone, authorizedContacts | Not present in basic flow         | Required                            |
| Directors              | directors                        | Not provided in basic flow        | corporateDirectors[] (optional/required depending on business type) |
| Response ID            | onboardingId                     | applicationId                     | data.entityId                       |
| Response URL           | onboardingForm                   | url                               | Not present in sample; may exist elsewhere |

***

### **Unified Approach**

- Use a superset data model for your onboarding form/service.
- Down-map data fields as needed per provider, abstracting away naming/structural differences.
- Normalize responses so all your workflows expect: `{ "onboardingId": "...", "onboardingUrl": "..." }`
- Handle provider-specific optional fields with optional chaining/defaults in your backend.

***

**In summary:**  
You can build a single onboarding workflow, but must clearly handle provider-specific required/optional fields and carefully map requests and responses. The **delta** is mostly in Monex expecting more information up-front, and Nium returning an onboarding form URL. Both can be unified cleanly using an abstraction/service pattern, letting you onboard corporates to either provider with minimal maintenance and maximum data fidelity.

Sources
[1] create https://sbox.monexusa.com/api/entities/create

