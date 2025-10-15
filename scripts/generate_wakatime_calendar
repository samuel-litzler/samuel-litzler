#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import sys
import json
import time
import base64
import argparse
import datetime as dt
from urllib import request, error
import html

# Jours sans code (couleur neutre, hors palette des 6 paliers)
ZERO_COLOR = "#fafafa"   # très clair
ZERO_LABEL = "0h"

# -------------------------
# CLI
# -------------------------

def parse_args():
    p = argparse.ArgumentParser(description="Génère un calendrier (heatmap) WakaTime en SVG.")
    g = p.add_mutually_exclusive_group()
    g.add_argument("--last-days", type=int, default=365, help="Nombre de jours à couvrir (défaut: 365).")
    g.add_argument("--year", type=int, help="Année civile à représenter (ex: 2025).")

    p.add_argument("--timezone", default=os.getenv("TZ", "Europe/Paris"), help="Fuseau horaire WakaTime (défaut: Europe/Paris).")
    p.add_argument("--output", default="wakatime_calendar.svg", help="Chemin du fichier SVG de sortie.")
    p.add_argument("--title", default=None, help="Titre affiché en haut du SVG.")
    p.add_argument("--api-key", default=os.getenv("WAKATIME_API_KEY"), help="Clé API WakaTime (ou variable d'env).")
    p.add_argument("--max-retries", type=int, default=5, help="Tentatives réseau (défaut 5).")
    p.add_argument("--timeout", type=int, default=60, help="Timeout HTTP en secondes (défaut 60).")
    p.add_argument("--palette", default="green6", choices=["green6", "blue6", "orange6", "purple6", "gray6"], help="Palette (6 niveaux).")
    return p.parse_args()

def compute_period(args):
    today = dt.date.today()
    if args.year:
        start = dt.date(args.year, 1, 1)
        end = dt.date(args.year, 12, 31)
    else:
        end = today
        start = end - dt.timedelta(days=max(1, args.last_days) - 1)
    return start, end

# -------------------------
# Palette (6 couleurs)
# -------------------------

def build_palette6(name):
    # 6 teintes du plus clair au plus foncé
    if name == "blue6":
        return ["#eef5ff", "#d6e7ff", "#b8d5ff", "#8bb8ff", "#3f8cff", "#0b61ff"]
    if name == "orange6":
        return ["#fff3e8", "#ffe0c7", "#ffc191", "#ff9b57", "#ef6c00", "#b34f00"]
    if name == "purple6":
        return ["#f4efff", "#e1d5ff", "#c2a8ff", "#9b78ff", "#6a38c9", "#4d28a1"]
    if name == "gray6":
        return ["#eeeeee", "#dddddd", "#c9c9c9", "#b0b0b0", "#8b8b8b", "#616161"]
    # default green
    return ["#f0f5ee", "#d7eec6", "#b2e08b", "#7bc96f", "#2ea043", "#196127"]

# -------------------------
# Buckets (6 niveaux)
# -------------------------
# 0: <1h, 1: 1–3h59, 2: 4–8h59, 3: 9–10h59, 4: 11–12h59, 5: >=13h
def bucket6(seconds):
    if seconds < 3600:            # < 1h
        return 0
    if seconds < 4 * 3600:        # 1h–3h59
        return 1
    if seconds < 9 * 3600:        # 4h–8h59
        return 2
    if seconds < 11 * 3600:       # 9h–10h59
        return 3
    if seconds < 13 * 3600:       # 11h–12h59
        return 4
    return 5                      # >= 13h

# Libellés pour la légende
LEGEND_LABELS = ["< 1h", "1–3h", "4–8h", "9–10h", "11–12h", "≥ 13h"]

# -------------------------
# API
# -------------------------

def fetch_summaries(api_key, start, end, tz, timeout=60, max_retries=5):
    if not api_key or not api_key.startswith("waka_"):
        raise RuntimeError("Clé API WakaTime absente/invalide. Utilise --api-key ou WAKATIME_API_KEY.")
    auth = "Basic " + base64.b64encode(f"{api_key}:".encode()).decode()
    url = (
        "https://wakatime.com/api/v1/users/current/summaries"
        f"?start={start.isoformat()}&end={end.isoformat()}&timezone={tz}"
    )
    headers = {"Authorization": auth, "Accept": "application/json", "User-Agent": "WakaCalendar/1.1"}
    req = request.Request(url, headers=headers)

    last_err = None
    for i in range(1, max_retries + 1):
        try:
            with request.urlopen(req, timeout=timeout) as resp:
                return json.loads(resp.read().decode())["data"]
        except (error.HTTPError, error.URLError) as e:
            last_err = e
            status = getattr(e, "code", None)
            if status in (429, 500, 502, 503, 504) or "Remote end closed" in str(e):
                time.sleep(2 * i)
                continue
            raise
    raise last_err

# -------------------------
# SVG
# -------------------------

def build_heatmap_svg(totals, start, end, tz, palette, title=None):
    import xml.sax.saxutils as sx

    # Grille type GitHub : colonnes = semaines, lignes = jours (lundi→dimanche)
    start_monday = start - dt.timedelta(days=start.weekday())
    end_sunday = end + dt.timedelta(days=(6 - end.weekday()))

    # Layout
    title_top = 16                 # position du titre
    title_bottom_pad = 14          # padding sous le titre
    month_labels_y = title_top + title_bottom_pad
    top = month_labels_y + 14      # début de la grille
    left = 48
    cell, gap = 10, 2
    col_w = cell + 16              # largeur par item de légende

    days = (end_sunday - start_monday).days + 1
    weeks = (days + 6) // 7
    base_width = left + weeks * (cell + gap) + 36
    # Légende = 1 couleur "0h" + 6 couleurs de palette
    legend_colors = [ZERO_COLOR] + palette
    legend_labels = [ZERO_LABEL] + LEGEND_LABELS
    legend_needed_width = left + len(legend_colors) * col_w + 20
    width = max(base_width, legend_needed_width)

    height = top + 7 * (cell + gap) + 56   # + espace légende

    # Titre
    if not title:
        title = f"WakaTime • Activité quotidienne ({start.isoformat()} → {end.isoformat()}, TZ {tz})"
    title_esc = sx.escape(title)

    # Repères de mois (échappés)
    month_svg = []
    last_month = None
    for w in range(weeks):
        week_start = start_monday + dt.timedelta(days=w * 7)
        if week_start.month != last_month:
            last_month = week_start.month
            x = left + w * (cell + gap)
            month_svg.append(
                f'<text x="{x}" y="{month_labels_y}" font-size="10" fill="#666">{sx.escape(week_start.strftime("%b"))}</text>'
            )

    # Cases (fill différent si 0 seconde)
    rects = []
    cur = start_monday
    for w in range(weeks):
        for d in range(7):
            date_str = cur.isoformat()
            s = totals.get(date_str, 0)
            if s <= 0:
                fill = ZERO_COLOR
            else:
                b = bucket6(s)
                fill = palette[b]

            x = left + w * (cell + gap)
            y = top + d * (cell + gap)
            hours = int(s // 3600)
            mins = int((s % 3600) // 60)
            tooltip = sx.escape(f"{date_str} • {hours}h{mins:02d}")
            rects.append(
                f'<rect x="{x}" y="{y}" width="{cell}" height="{cell}" rx="2" ry="2" '
                f'fill="{fill}"><title>{tooltip}</title></rect>'
            )
            cur += dt.timedelta(days=1)

    # Légende (0h en premier, puis les 6 paliers)
    legend_top_rect = height - 30
    legend_top_text = height - 12
    legend_svg = []
    for i, color in enumerate(legend_colors):
        x = left + i * col_w
        legend_svg.append(
            f'<rect x="{x}" y="{legend_top_rect}" width="{cell}" height="{cell}" rx="2" ry="2" fill="{color}"/>'
        )
        legend_svg.append(
            f'<text x="{x + cell/2}" y="{legend_top_text}" font-size="5" fill="#555" text-anchor="middle">{sx.escape(legend_labels[i])}</text>'
        )

    # CSS en CDATA pour éviter les erreurs XML
    style_block = """<![CDATA[
      text { font-family: -apple-system, BlinkMacSystemFont, Segoe UI, Helvetica, Arial, sans-serif; }
    ]]>"""

    svg = (
        f'<svg xmlns="http://www.w3.org/2000/svg" width="{width}" height="{height}" '
        f'viewBox="0 0 {width} {height}">\n'
        f'  <style type="text/css">{style_block}</style>\n'
        f'  <text x="{left}" y="{title_top}" font-size="12" fill="#333">{title_esc}</text>\n'
        f'  {"".join(month_svg)}\n'
        f'  {"".join(rects)}\n'
        f'  {"".join(legend_svg)}\n'
        f'</svg>'
    )
    return svg

# -------------------------
# Main
# -------------------------

def main():
    args = parse_args()
    api_key = args.api_key
    if not api_key or not api_key.startswith("waka_"):
        print("❌ Clé API manquante ou invalide. Passe-la via --api-key ou WAKATIME_API_KEY.", file=sys.stderr)
        sys.exit(1)

    start, end = compute_period(args)
    palette = build_palette6(args.palette)

    try:
        summaries = fetch_summaries(api_key, start, end, args.timezone, args.timeout, args.max_retries)
    except Exception as e:
        print(f"❌ Erreur lors de l'appel WakaTime: {e}", file=sys.stderr)
        sys.exit(2)

    totals = {d["range"]["date"]: d["grand_total"]["total_seconds"] for d in summaries}
    svg = build_heatmap_svg(totals, start, end, args.timezone, palette, title=args.title)

    with open(args.output, "w", encoding="utf-8") as f:
        f.write(svg)

    print(f"✅ SVG généré: {args.output}")
    print(f"   Période: {start} -> {end} | TZ: {args.timezone} | Palette: {args.palette}")

if __name__ == "__main__":
    main()
