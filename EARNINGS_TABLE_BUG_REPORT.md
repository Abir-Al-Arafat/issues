# Bug Report: Earnings Table Pagination & Duplication Issue

## 🚨 The Problem
When hitting the `{{baseURLLocal}}/dashboard/earnings-table` API endpoint, the response was exhibiting a mismatch between the actual data payload and the pagination metadata. 

Specifically, the `pagination.totalItems` correctly reported `3` items, but the `data.items` array contained `6` objects. Upon inspecting the payload, the 3 unique ride records were perfectly duplicated (each `id` appeared exactly twice).

## 🔍 Why It Happened (The Root Cause)
The issue stemmed from the MongoDB Aggregation Pipeline inside the `rideRepo.getEarningsTableData` method, specifically within the `$facet` stage where the paginated data is resolved.

In MongoDB, when you perform a `$lookup` (join) on a collection that returns multiple matches, it attaches an array of those matched documents to your base document. If you then apply an `$unwind` operation to that array, MongoDB flattens it by creating a separate copy of the parent document for *every* item in the array.

**The Culprit: The Fare Rules Join**
```typescript
// Old Code (Caused Duplication)
{
  $lookup: {
    from: "farerules",
    localField: "driverDetails.gender",
    foreignField: "gender",
    as: "fareRuleDetails",
  },
},
{
  $unwind: {
    path: "$fareRuleDetails",
    preserveNullAndEmptyArrays: true,
  },
}
```
Because there were multiple fare rules in the database matching the driver's gender, the `$lookup` pulled in an array of two fare rules. The `$unwind` operator then forcefully split each single ride document into two identical ride documents. `3` unique rides × `2` fare rules = `6` duplicate output items. 

The `totalCount` remained correct at `3` because the metadata counting facet executed completely separate from the data formatting pipeline.

## ✅ The Solution
To fix this, we needed to prevent the `$lookup` from returning multiple documents in the first place. We achieved this by converting the standard `$lookup` syntax into a **sub-pipeline** `$lookup`. 

By adding a `$match` operator combined with a `{ $limit: 1 }` operator inside the lookup's pipeline, we enforce a strict one-to-one join. This guarantees that the `$unwind` operator receives an array with a maximum of one item, completely eliminating the duplication of the parent ride records.

**The Fixed Code:**
```typescript
// New Code (Fixed)
{
  $lookup: {
    from: "farerules",
    let: { driverGender: "$driverDetails.gender" },
    pipeline: [
      {
        $match: {
          $expr: { $eq: ["$gender", "$$driverGender"] },
        },
      },
      // { $match: { isActive: true } }, // Optional: highly recommended if rules have active states
      { $limit: 1 }, // <-- This strict limit stops the duplication
    ],
    as: "fareRuleDetails",
  },
},
{
  $unwind: {
    path: "$fareRuleDetails",
    preserveNullAndEmptyArrays: true,
  },
}
```

### 🎯 Result
The `items` array now accurately reflects exactly `3` unique records, perfectly matching the `totalItems: 3` pagination metadata.
