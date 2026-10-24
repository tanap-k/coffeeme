#!/usr/bin/env python3
"""Build cafes-th.json for CoffeeMe from an osmium GeoJSON-sequence export.

Input : cafes.geojsonseq  (from `osmium export ... -f geojsonseq`)
Output: cafes-th.json     (compact list the website loads from its own domain)

Each cafe row is: [lat, lng, name, address, brandText]
  - brandText = brand / operator / other-language names, used by the website
    to hide big chains (Cafe Amazon, Starbucks, ...). Empty string if none.

Data (c) OpenStreetMap contributors, ODbL 1.0 - https://www.openstreetmap.org/copyright
"""
import json
import sys
from datetime import datetime, timezone

SRC = sys.argv[1] if len(sys.argv) > 1 else "cafes.geojsonseq"
OUT = sys.argv[2] if len(sys.argv) > 2 else "cafes-th.json"


def mean_point(coords):
    """Average of a list of [lng, lat] pairs -> (lat, lng)."""
    if not coords:
        return None
    lng = sum(c[0] for c in coords) / len(coords)
    lat = sum(c[1] for c in coords) / len(coords)
    return lat, lng


def representative_point(geom):
    t = geom.get("type")
    c = geom.get("coordinates")
    if not c:
        return None
    if t == "Point":
        return c[1], c[0]
    if t in ("LineString", "MultiPoint"):
        return mean_point(c)
    if t == "Polygon":
        return mean_point(c[0])          # outer ring
    if t == "MultiLineString":
        return mean_point(c[0])
    if t == "MultiPolygon":
        return mean_point(c[0][0])       # outer ring of first polygon
    return None


def address_of(p):
    parts = [p.get(k) for k in ("addr:housenumber", "addr:street", "addr:subdistrict",
                                "addr:district", "addr:city", "addr:province")]
    seen, out = set(), []
    for x in parts:
        if x and x not in seen:
            seen.add(x)
            out.append(x)
    return " ".join(out)


def main():
    rows, seen_ids = [], set()
    with open(SRC, encoding="utf-8") as f:
        for line in f:
            line = line.strip().lstrip("\x1e").strip()   # geojsonseq record separator
            if not line:
                continue
            feat = json.loads(line)
            p = feat.get("properties") or {}
            fid = (p.get("@type"), p.get("@id"))
            if fid[1] is not None:
                if fid in seen_ids:
                    continue
                seen_ids.add(fid)
            pt = representative_point(feat.get("geometry") or {})
            if not pt:
                continue
            lat, lng = pt
            name = p.get("name") or p.get("name:th") or p.get("name:en") or ""
            brand_bits = [p.get(k) for k in ("brand", "brand:en", "brand:th", "operator", "name:en", "name:th")]
            uniq = []
            for b in brand_bits:
                if b and b != name and b not in uniq:
                    uniq.append(b)
            brand_text = " | ".join(uniq)
            rows.append([round(lat, 5), round(lng, 5), name, address_of(p), brand_text])

    rows.sort(key=lambda r: (r[0], r[1]))
    data = {
        "generated": datetime.now(timezone.utc).strftime("%Y-%m-%d"),
        "source": "OpenStreetMap via Geofabrik (thailand-latest.osm.pbf)",
        "license": "Data (c) OpenStreetMap contributors, ODbL 1.0",
        "fields": ["lat", "lng", "name", "address", "brandText"],
        "cafes": rows,
    }
    with open(OUT, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, separators=(",", ":"))
    print(f"wrote {OUT}: {len(rows)} cafes")
    if len(rows) < 100:
        # Thailand has thousands of cafes in OSM; a tiny result means something broke.
        sys.exit("too few cafes - refusing to publish a broken data file")


if __name__ == "__main__":
    main()
