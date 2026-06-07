# Maldives Prayer Times

A minimal, self-contained MySQL dataset of daily Islamic prayer times for the Maldives, designed for **prayer-times-by-coordinates** lookups.

Given a device's latitude/longitude, you find the nearest island that has prayer data and read today's times.

## What's in the dataset

`maldives_prayer_times.sql` creates two tables:

| Table | Rows | Purpose |
|-------|------|---------|
| `tbl_islands` | 202 | Islands that have prayer data — `lat`/`lng`, their latitude band (`prayer_table_id`), and a per-island minute offset (`prayer_adjust_mins`) |
| `tbl_prayer_times` | 15,372 | 42 latitude bands × 366 days of prayer times |

### Two design details worth knowing

1. **Perpetual calendar.** `tbl_prayer_times.date` uses a **dummy year `0004`** (a leap year, so Feb 29 exists). Prayer times barely drift year to year, so one canonical year is reused forever. **Match on month + day, never on the full date:**

   ```sql
   WHERE MONTH(`date`) = MONTH(CURDATE()) AND DAY(`date`) = DAY(CURDATE())
   ```

2. **Per-island offset.** Prayer times are computed per **latitude band**, but each island carries a small `prayer_adjust_mins` correction. **Add it to every prayer time** or your results can be off by a minute:

   ```sql
   ADDTIME(t.fajr, SEC_TO_TIME(COALESCE(i.prayer_adjust_mins, 0) * 60))
   ```

## Setup

```bash
mysql -u root -p -e "CREATE DATABASE maldives_prayer_times"
mysql -u root -p maldives_prayer_times < maldives_prayer_times.sql
```

## Usage

### Get today's prayer times for the nearest island to a coordinate

Pass the device coordinate as `:lat` / `:lng`. The query finds the closest island with prayer data, then returns today's times with the per-island offset applied.

```sql
SELECT
  CURDATE()        AS today,
  n.name_en        AS island,
  n.reg_no,
  ADDTIME(t.fajr,    SEC_TO_TIME(n.adj * 60)) AS fajr,
  ADDTIME(t.sunrise, SEC_TO_TIME(n.adj * 60)) AS sunrise,
  ADDTIME(t.duhr,    SEC_TO_TIME(n.adj * 60)) AS duhr,
  ADDTIME(t.asr,     SEC_TO_TIME(n.adj * 60)) AS asr,
  ADDTIME(t.maghrib, SEC_TO_TIME(n.adj * 60)) AS maghrib,
  ADDTIME(t.isha,    SEC_TO_TIME(n.adj * 60)) AS isha
FROM (
  SELECT name_en, reg_no, prayer_table_id,
         COALESCE(prayer_adjust_mins, 0) AS adj
  FROM tbl_islands
  ORDER BY POW(lat - :lat, 2)
         + POW((lng - :lng) * COS(RADIANS(:lat)), 2)   -- nearest island (cheap planar distance, lng scaled by cos lat)
  LIMIT 1
) n
JOIN tbl_prayer_times t
  ON t.prayer_table_id = n.prayer_table_id
 AND MONTH(t.`date`) = MONTH(CURDATE())
 AND DAY(t.`date`)   = DAY(CURDATE());
```

The distance expression is squared planar distance with longitude scaled by `COS(RADIANS(:lat))`. For nearest-neighbour ranking within a small country this gives the same ordering as great-circle distance but is far cheaper (no trig per row). If you also want to *show* the distance, compute haversine on the single winning row only.

### Get today's prayer times for a specific island (by registration number)

```sql
SELECT
  CURDATE() AS today,
  i.name_en AS island,
  ADDTIME(t.fajr,    SEC_TO_TIME(COALESCE(i.prayer_adjust_mins,0)*60)) AS fajr,
  ADDTIME(t.sunrise, SEC_TO_TIME(COALESCE(i.prayer_adjust_mins,0)*60)) AS sunrise,
  ADDTIME(t.duhr,    SEC_TO_TIME(COALESCE(i.prayer_adjust_mins,0)*60)) AS duhr,
  ADDTIME(t.asr,     SEC_TO_TIME(COALESCE(i.prayer_adjust_mins,0)*60)) AS asr,
  ADDTIME(t.maghrib, SEC_TO_TIME(COALESCE(i.prayer_adjust_mins,0)*60)) AS maghrib,
  ADDTIME(t.isha,    SEC_TO_TIME(COALESCE(i.prayer_adjust_mins,0)*60)) AS isha
FROM tbl_islands i
JOIN tbl_prayer_times t
  ON t.prayer_table_id = i.prayer_table_id
WHERE i.reg_no = 'T10'                       -- Male'
  AND MONTH(t.`date`) = MONTH(CURDATE())
  AND DAY(t.`date`)   = DAY(CURDATE());
```

## Notes

- **Timezone.** `CURDATE()` uses the MySQL server's clock. The Maldives is UTC+5 with no DST. If your server runs in UTC, pass an explicit date instead of `CURDATE()` so the day boundary is correct at local midnight.
- Schema is InnoDB / `utf8mb4`; `tbl_prayer_times` is keyed on `(prayer_table_id, date)`.

## Schema

```sql
CREATE TABLE tbl_islands (
  island_id          smallint unsigned NOT NULL,
  reg_no             varchar(4)        DEFAULT NULL,
  name_en            varchar(255)      NOT NULL,
  lat                decimal(9,6)      NOT NULL,
  lng                decimal(9,6)      NOT NULL,
  prayer_table_id    smallint unsigned NOT NULL,
  prayer_adjust_mins tinyint           NOT NULL DEFAULT 0,
  PRIMARY KEY (island_id),
  KEY idx_table (prayer_table_id)
);

CREATE TABLE tbl_prayer_times (
  prayer_table_id smallint unsigned NOT NULL,
  `date`          date NOT NULL,
  fajr time NOT NULL, sunrise time NOT NULL, duhr time NOT NULL,
  asr  time NOT NULL, maghrib time NOT NULL, isha time NOT NULL,
  PRIMARY KEY (prayer_table_id, `date`)
);
```
