---
description: Who is hiring for a role across several cities, with salaries where employers publish them
---

Map the hiring for one role.

Ask me for the role and the cities if I have not given them. Three to five cities keeps the answer readable.

Then:

1. For each city call `hasdata_indeed_listing_getJobListings` with the role in `keyword`, the city in `location` and `sort: "date"`, so recent postings come first.
2. Per city report how many openings came back, the employers posting the most, and the salary range across the postings that carry one.
3. Say how many postings published a salary and how many did not. A range built on four of forty postings is a weak signal, and the reader deserves to know that.
4. List the ten most recent openings across all cities with title, company, city, salary where present and the posting date.
5. Note any employer that appears in more than one city, since that is usually the interesting pattern.

Do not open individual postings unless I ask for a description. If a city returns an empty list, say so plainly instead of padding the table with another city's rows.
