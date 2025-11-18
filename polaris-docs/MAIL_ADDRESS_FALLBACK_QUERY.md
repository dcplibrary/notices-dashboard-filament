# ENHANCED MAIL QUERY WITH ADDRESS FALLBACK

**Purpose:** Handle cases where patrons may not have a "Notice" address but have a "Mailing" address
**Use:** Optional enhancement for Query 2 (Mail Notifications Export)

---

## QUERY: MAIL NOTIFICATIONS WITH ADDRESS FALLBACK

This enhanced version tries **AddressTypeID = 2 (Notice)** first, then falls back to **AddressTypeID = 12 (Mailing)** if Notice address is not available.

### SQL Query

```sql
SET NOCOUNT ON

SELECT
    -- Notification Log Fields
    nl.NotificationLogID,
    nl.PatronID,
    nl.NotificationDateTime,
    nl.NotificationTypeID,
    nl.DeliveryOptionID,
    nl.NotificationStatusID,
    nl.OverduesCount,
    nl.HoldsCount,
    nl.CancelsCount,
    nl.BillsCount,
    nl.ManualBillCount,
    nl.Overdues2ndCount,
    nl.Overdues3rdCount,
    nl.RoutingsCount,
    nl.RecallsCount,
    nl.ReportingOrgID,
    nl.LanguageID,
    nl.Details,

    -- Patron Information
    nl.PatronBarcode,
    pr.NameFirst,
    pr.NameLast,
    pr.NameMiddle,
    pr.PatronFullName,

    -- Mailing Address (with fallback logic)
    -- Try Notice address first, then Mailing address
    COALESCE(a_notice.StreetOne, a_mailing.StreetOne) AS StreetOne,
    COALESCE(a_notice.StreetTwo, a_mailing.StreetTwo) AS StreetTwo,
    COALESCE(a_notice.StreetThree, a_mailing.StreetThree) AS StreetThree,
    COALESCE(a_notice.MunicipalityName, a_mailing.MunicipalityName) AS MunicipalityName,
    COALESCE(a_notice.ZipPlusFour, a_mailing.ZipPlusFour) AS ZipPlusFour,
    COALESCE(pc_notice.PostalCode, pc_mailing.PostalCode) AS PostalCode,
    COALESCE(pc_notice.City, pc_mailing.City) AS City,
    COALESCE(pc_notice.State, pc_mailing.State) AS State,
    COALESCE(pc_notice.County, pc_mailing.County) AS County,
    COALESCE(c_notice.CountryName, c_mailing.CountryName) AS CountryName,

    -- Address Metadata (which address type was used)
    CASE
        WHEN a_notice.AddressID IS NOT NULL THEN 2  -- Notice address
        WHEN a_mailing.AddressID IS NOT NULL THEN 12  -- Mailing address
        ELSE NULL
    END AS AddressTypeUsed,

    CASE
        WHEN a_notice.AddressID IS NOT NULL THEN 'Notice'
        WHEN a_mailing.AddressID IS NOT NULL THEN 'Mailing'
        ELSE 'No Address'
    END AS AddressTypeDescription,

    -- Verification info from Notice address (preferred)
    COALESCE(pa_notice.Verified, pa_mailing.Verified) AS AddressVerified,
    COALESCE(pa_notice.VerificationDate, pa_mailing.VerificationDate) AS AddressVerificationDate,

    -- Lookup Descriptions
    ns.Description AS NotificationStatusDescription,
    nt.Description AS NotificationTypeDescription

FROM
    Polaris.Polaris.NotificationLog nl (NOLOCK)

    -- Join to get patron info
    LEFT JOIN Polaris.Polaris.Patrons p (NOLOCK)
        ON nl.PatronID = p.PatronID
    LEFT JOIN Polaris.Polaris.PatronRegistration pr (NOLOCK)
        ON nl.PatronID = pr.PatronID

    -- Join to get NOTICE address (AddressTypeID = 2) - FIRST CHOICE
    LEFT JOIN Polaris.Polaris.PatronAddresses pa_notice (NOLOCK)
        ON nl.PatronID = pa_notice.PatronID
        AND pa_notice.AddressTypeID = 2  -- Notice address
    LEFT JOIN Polaris.Polaris.Addresses a_notice (NOLOCK)
        ON pa_notice.AddressID = a_notice.AddressID
    LEFT JOIN Polaris.Polaris.PostalCodes pc_notice (NOLOCK)
        ON a_notice.PostalCodeID = pc_notice.PostalCodeID
    LEFT JOIN Polaris.Polaris.Countries c_notice (NOLOCK)
        ON pc_notice.CountryID = c_notice.CountryID

    -- Join to get MAILING address (AddressTypeID = 12) - FALLBACK
    LEFT JOIN Polaris.Polaris.PatronAddresses pa_mailing (NOLOCK)
        ON nl.PatronID = pa_mailing.PatronID
        AND pa_mailing.AddressTypeID = 12  -- Mailing address
    LEFT JOIN Polaris.Polaris.Addresses a_mailing (NOLOCK)
        ON pa_mailing.AddressID = a_mailing.AddressID
    LEFT JOIN Polaris.Polaris.PostalCodes pc_mailing (NOLOCK)
        ON a_mailing.PostalCodeID = pc_mailing.PostalCodeID
    LEFT JOIN Polaris.Polaris.Countries c_mailing (NOLOCK)
        ON pc_mailing.CountryID = c_mailing.CountryID

    -- Join to get status and type descriptions
    LEFT JOIN Polaris.Polaris.NotificationStatuses ns (NOLOCK)
        ON nl.NotificationStatusID = ns.NotificationStatusID
    LEFT JOIN Polaris.Polaris.NotificationTypes nt (NOLOCK)
        ON nl.NotificationTypeID = nt.NotificationTypeID

WHERE
    nl.DeliveryOptionID = 1  -- Mail only
    AND nl.NotificationDateTime >= DATEADD(day, -7, GETDATE())  -- Last 7 days
    AND p.RecordStatusID = 1  -- Active patrons only

ORDER BY
    nl.NotificationDateTime DESC,
    nl.PatronBarcode
```

---

## HOW FALLBACK WORKS

### Address Priority Logic

```
1. Try AddressTypeID = 2 (Notice)
   ↓ (if NULL)
2. Try AddressTypeID = 12 (Mailing)
   ↓ (if NULL)
3. All address fields will be NULL
```

### Example Results

**Scenario 1: Patron has Notice address**
```
StreetOne: "123 Main St"  (from Notice address)
AddressTypeUsed: 2
AddressTypeDescription: "Notice"
```

**Scenario 2: Patron has no Notice address but has Mailing address**
```
StreetOne: "456 Oak Ave"  (from Mailing address)
AddressTypeUsed: 12
AddressTypeDescription: "Mailing"
```

**Scenario 3: Patron has neither Notice nor Mailing address**
```
StreetOne: NULL
AddressTypeUsed: NULL
AddressTypeDescription: "No Address"
```

---

## PERFORMANCE NOTES

### Pros
- ✅ More complete data (fewer NULL addresses)
- ✅ Better coverage of patrons
- ✅ Clear indication of which address type was used

### Cons
- ⚠️ More complex query (2x address joins)
- ⚠️ Slightly slower execution (more joins)
- ⚠️ Larger result set in memory

### Recommendation

**Use this enhanced query if:**
- Many patrons lack Notice addresses
- You need maximum address coverage
- Performance impact is acceptable

**Use simple query (AddressTypeID = 2 only) if:**
- Most patrons have Notice addresses configured
- Performance is critical
- You want to flag missing Notice addresses

---

## VERIFY YOUR DATA DISTRIBUTION

Before choosing which query to use, check how many patrons have each address type:

```sql
-- Count patrons by address type
SELECT
    at.AddressTypeID,
    at.Description,
    COUNT(DISTINCT pa.PatronID) AS PatronCount
FROM
    Polaris.Polaris.PatronAddresses pa
    JOIN Polaris.Polaris.AddressTypes at
        ON pa.AddressTypeID = at.AddressTypeID
WHERE
    pa.AddressTypeID IN (2, 12)  -- Notice and Mailing
GROUP BY
    at.AddressTypeID,
    at.Description
ORDER BY
    PatronCount DESC
```

**Expected Output:**
```
AddressTypeID | Description | PatronCount
2             | Notice      | 15,234
12            | Mailing     | 8,456
```

**Decision Guide:**
- If **Notice (2)** has most patrons → Use simple query
- If **Mailing (12)** has many more patrons → Use fallback query or switch primary to Mailing
- If roughly equal → Use fallback query for maximum coverage

---

## ALTERNATIVE: GENERIC ADDRESS FALLBACK

If you want even broader coverage, you could also add **AddressTypeID = 1 (Generic)** as a third fallback:

```sql
-- Triple fallback: Notice → Mailing → Generic
COALESCE(a_notice.StreetOne, a_mailing.StreetOne, a_generic.StreetOne) AS StreetOne
```

But this adds even more complexity and may pull addresses not intended for mailings.

---

**Status:** Optional Enhancement
**Recommended:** Test data distribution first, then decide
**Last Updated:** November 18, 2025
