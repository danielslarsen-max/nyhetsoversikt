#!/usr/bin/env python3
"""Nyhetsoversikt: henter RSS fra norske aviser, grupperer like saker og rangerer dem.
Kjør:  python3 nyhetsoversikt.py   -> lager nyheter.html og åpner den i nettleseren.
Rangering = antall aviser som dekker saken (viktigst) + ferskhet + plassering i avisens feed."""
import re, math, html, webbrowser, urllib.request
import xml.etree.ElementTree as ET
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from concurrent.futures import ThreadPoolExecutor

# Rediger fritt. Feeder som ikke svarer hoppes over. (Adressene er ikke verifisert av meg.)
FEEDS = {
    "NRK": "https://www.nrk.no/toppsaker.rss",
    "VG": "https://www.vg.no/rss/feed",
    "Dagbladet": "https://www.dagbladet.no/rss",
    "Aftenposten": "https://www.aftenposten.no/rss",
    "DN": "https://www.dn.no/rss",
    "E24": "https://e24.no/rss2/",
    "Nettavisen": "https://www.nettavisen.no/service/rss",
    "TV 2": "https://www.tv2.no/rss/nyheter",
    "Dagsavisen": "https://www.dagsavisen.no/rss",
    "Adresseavisen": "https://www.adressa.no/rss",
    "BT": "https://www.bt.no/rss",
    "Aftenbladet": "https://www.aftenbladet.no/rss",
    "Fædrelandsvennen": "https://www.fvn.no/rss",
}
MAX_SAKER = 60          # antall saker i listen
BILDER_PÅ_TOPP = 5      # bare de viktigste sakene får bilde
LIKHET = 0.34           # hvor like overskrifter må være for å regnes som samme sak
STOPP = set("og i på av til for med som er det en et den de har ble blir fra om at ikke var etter over under mot men også kan skal vil han hun seg sin sitt".split())

def ord_av(t):
    return {w for w in re.findall(r"\w+", t.lower()) if len(w) > 3 and w not in STOPP}

def lokal(tag):
    return tag.rsplit("}", 1)[-1]

def hent(item):
    navn, url = item
    try:
        req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0 nyhetsoversikt"})
        rot = ET.fromstring(urllib.request.urlopen(req, timeout=12).read())
    except Exception as e:
        print(f"  hopper over {navn}: {e}")
        return []
    ut = []
    for el in rot.iter():
        if lokal(el.tag) not in ("item", "entry"):
            continue
        d = {"kilde": navn, "tittel": "", "lenke": "", "tid": None, "bilde": ""}
        for c in el:
            t = lokal(c.tag)
            if t == "title": d["tittel"] = html.unescape((c.text or "").strip())
            elif t == "link": d["lenke"] = (c.text or c.get("href") or "").strip()
            elif t in ("pubDate", "published", "updated", "date") and not d["tid"]:
                try: d["tid"] = parsedate_to_datetime(c.text)
                except Exception:
                    try: d["tid"] = datetime.fromisoformat((c.text or "").replace("Z", "+00:00"))
                    except Exception: pass
            elif t in ("content", "thumbnail", "enclosure") and c.get("url") and "image" in (c.get("type") or "image"):
                d["bilde"] = d["bilde"] or c.get("url")
        if d["tittel"] and d["lenke"]:
            ut.append(d)
    for i, d in enumerate(ut):
        d["pos"] = 1 - i / max(len(ut), 1)
    print(f"  {navn}: {len(ut)} saker")
    return ut

def main():
    print("Henter nyheter ...")
    with ThreadPoolExecutor(8) as ex:
        alle = [x for liste in ex.map(hent, FEEDS.items()) for x in liste]
    nå = datetime.now(timezone.utc)
    grupper = []
    for a in alle:
        a["ord"] = ord_av(a["tittel"])
        for g in grupper:
            if a["kilde"] in g["kilder"]:
                continue
            u = g["ord"] | a["ord"]
            if u and len(g["ord"] & a["ord"]) / len(u) >= LIKHET:
                g["art"].append(a); g["kilder"].add(a["kilde"]); g["ord"] |= a["ord"]
                break
        else:
            grupper.append({"art": [a], "kilder": {a["kilde"]}, "ord": set(a["ord"])})
    for g in grupper:
        tider = [a["tid"].astimezone(timezone.utc) for a in g["art"] if a["tid"]]
        g["nyest"] = max(tider) if tider else None
        alder_t = (nå - g["nyest"]).total_seconds() / 3600 if g["nyest"] else 24
        g["score"] = 3 * len(g["kilder"]) + 2 * math.exp(-max(alder_t, 0) / 12) + 2 * max(a["pos"] for a in g["art"])
        g["art"].sort(key=lambda a: -a["pos"])
    grupper.sort(key=lambda g: -g["score"])

    rader = []
    for n, g in enumerate(grupper[:MAX_SAKER]):
        a = g["art"][0]
        bilde = next((x["bilde"] for x in g["art"] if x["bilde"]), "") if n < BILDER_PÅ_TOPP else ""
        img = f'<img src="{html.escape(bilde)}" alt="">' if bilde else ""
        kl = " ".join(f'<a class="k" href="{html.escape(x["lenke"])}">{html.escape(x["kilde"])}</a>' for x in g["art"])
        kl_t = f'{g["nyest"].astimezone().strftime("%H:%M")} · ' if g["nyest"] else ""
        rader.append(f'<li>{img}<div><a class="t" href="{html.escape(a["lenke"])}">{html.escape(a["tittel"])}</a>'
                     f'<div class="m">{kl_t}{len(g["kilder"])} aviser: {kl}</div></div></li>')
    side = f"""<!doctype html><html lang="no"><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Dagens nyhetsbilde</title><style>
body{{font:16px/1.4 system-ui,sans-serif;max-width:760px;margin:2rem auto;padding:0 1rem;color:#222}}
ol{{padding:0;list-style:none;counter-reset:n}} li{{display:flex;gap:.8rem;padding:.55rem 0;border-bottom:1px solid #ddd;counter-increment:n}}
li::before{{content:counter(n);color:#999;width:1.6rem;flex:none;text-align:right}}
li img{{width:72px;height:48px;object-fit:cover;border-radius:3px;flex:none}}
.t{{color:inherit;font-weight:600;text-decoration:none}} .t:hover{{text-decoration:underline}}
.m{{font-size:.8rem;color:#777}} .k{{color:#555;margin-right:.3rem}}
@media(prefers-color-scheme:dark){{body{{background:#161616;color:#eee}}li{{border-color:#333}}.m,.k{{color:#aaa}}}}
</style><h1>Dagens nyhetsbilde</h1><p class="m">Oppdatert {datetime.now().strftime("%d.%m.%Y %H:%M")} · {len(alle)} saker fra {len(FEEDS)} aviser</p>
<ol>{''.join(rader)}</ol></html>"""
    with open("nyheter.html", "w", encoding="utf-8") as f:
        f.write(side)
    print("Ferdig: nyheter.html")
    webbrowser.open("nyheter.html")

if __name__ == "__main__":
    main()

