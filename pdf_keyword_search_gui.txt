# -*- coding: utf-8 -*-
"""
PDF Keyword Search Tool
Searches through all PDF files in a selected folder (and subfolders)
and highlights bullet-point lines containing a given keyword.
Generates an HTML report with all matches and opens it in the browser.
"""

import os
import re
import sys
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime
from html import escape

import fitz  # PyMuPDF

# --- GUI ---
import tkinter as tk
from tkinter import filedialog, messagebox

# ========= Utility Functions =========
# Accept common bullet characters (ASCII + Unicode)
BULLET_PREFIX_RE = re.compile(r"^\s*([-*•‣▪◦·‒–—])\s+")
SPACE_RE = re.compile(r"\s+")

def normalize_space(s: str) -> str:
    return SPACE_RE.sub(" ", s).strip()

def is_bullet_line(line: str) -> bool:
    l = line.lstrip()
    if not l:
        return False
    # First character determines a bullet
    first_char = l[:1]
    return first_char in ("-", "*", "•", "‣", "▪", "◦", "·", "‒", "–", "—")

def strip_bullet_prefix(line: str) -> str:
    return BULLET_PREFIX_RE.sub("", line).strip()

def join_with_hyphen_fix(prev: str, nxt: str) -> str:
    """Join wrapped lines, removing trailing hyphen splits."""
    prev = prev.rstrip()
    nxt = nxt.strip()
    if prev.endswith("-"):
        return prev[:-1] + nxt
    return prev + " " + nxt

def bullets_from_page_text(txt: str):
    """Extract bullet-paragraphs from raw page text."""
    lines = txt.splitlines()
    bullets = []
    current = None

    def flush():
        nonlocal current
        if current is not None:
            cur = normalize_space(current)
            if cur:
                bullets.append(cur)
            current = None

    for ln in lines:
        if is_bullet_line(ln):
            flush()
            current = strip_bullet_prefix(ln)
        else:
            if current is not None and ln.strip():
                current = join_with_hyphen_fix(current, ln)
    flush()
    return bullets

def highlight_html(text: str, keyword: str) -> str:
    """Highlight matches in yellow and bold (case-insensitive)."""
    # Escape once, then inject tags around matches
    safe_text = escape(text)
    pattern = re.compile(re.escape(keyword), re.IGNORECASE)
    return pattern.sub(
        lambda m: f"<b style='background-color:yellow;color:black;'>{escape(m.group(0))}</b>",
        safe_text,
    )

def fmt_datetime(ts: float) -> str:
    return datetime.fromtimestamp(ts).strftime("%Y-%m-%d %H:%M")

def search_pdf_file(pdf_path: Path, kw_lower: str, mtime: float):
    """Return list of hits dicts: file, page, bullet, mtime."""
    hits = []
    try:
        with fitz.open(pdf_path) as doc:
            for i, page in enumerate(doc):
                try:
                    text = page.get_text("text") or ""
                except Exception:
                    text = ""
                if not text:
                    continue
                for b in bullets_from_page_text(text):
                    if kw_lower in b.lower():
                        hits.append({
                            "file": str(pdf_path),
                            "page": i + 1,
                            "bullet": "- " + b,
                            "mtime": mtime,
                        })
    except Exception as e:
        # Print so it’s visible in console builds; keep GUI silent
        print(f"[WARN] Error reading {pdf_path}: {e}")
    return hits

def build_html(keyword: str, all_hits: list) -> str:
    """Build a simple HTML report."""
    all_hits.sort(key=lambda x: (-x["mtime"], x["file"].lower(), x["page"]))
    parts = [
        "<!DOCTYPE html><html><head><meta charset='utf-8'>",
        "<title>PDF Search Results</title>",
        "<style>",
        "body{font-family:Arial,Helvetica,sans-serif;max-width:1000px;margin:auto;padding:16px}",
        "hr{border:none;border-top:1px solid #ddd}",
        "code{background:#f4f4f4;padding:2px 4px;border-radius:3px}",
        "</style></head><body>",
        f"<h3>Keyword: '<b>{escape(keyword)}</b>' — {len(all_hits)} match(es)</h3>",
    ]

    last_file = None
    for h in all_hits:
        if h["file"] != last_file:
            last_file = h["file"]
            parts.append(
                f"<p><b>File:</b> {escape(last_file)}"
                f"<br><b>Modified:</b> {fmt_datetime(h['mtime'])}</p>"
            )
        parts.append(
            f"<p style='margin-left:20px;'><b>Page:</b> {h['page']}<br>"
            f"{highlight_html(h['bullet'], keyword)}</p>"
        )
        parts.append("<hr>")
    parts.append("</body></html>")
    return "".join(parts)

def run_search(folder_path: str, keyword: str, max_workers: int = None) -> str:
    folder = Path(folder_path)
    if not folder.exists():
        raise FileNotFoundError(f"Folder not found: {folder}")

    # Collect PDFs (recursively)
    pdfs = []
    for p in folder.rglob("*.pdf"):
        try:
            mtime = p.stat().st_mtime
        except Exception:
            continue
        pdfs.append((p, mtime))

    if not pdfs:
        return build_html(keyword, [])

    if max_workers is None:
        max_workers = min(8, os.cpu_count() or 4)

    kw_lower = keyword.lower()
    all_hits = []
    with ThreadPoolExecutor(max_workers=max_workers) as ex:
        futures = {ex.submit(search_pdf_file, p, kw_lower, mtime): (p, mtime) for p, mtime in pdfs}
        for fut in as_completed(futures):
            try:
                all_hits.extend(fut.result())
            except Exception as e:
                pdf_path, _ = futures[fut]
                print(f"[WARN] Error processing {pdf_path}: {e}")

    return build_html(keyword, all_hits)

# ========= GUI =========
def launch_gui():
    import webbrowser

    root = tk.Tk()
    root.title("PDF Keyword Search Tool")
    root.geometry("720x250")

    frm = tk.Frame(root, padx=12, pady=12)
    frm.pack(fill="both", expand=True)

    out_path = os.path.abspath("pdf_search_results.html")

    # Folder selection
    tk.Label(frm, text="Folder to search:").grid(row=0, column=0, sticky="w")
    folder_entry = tk.Entry(frm, width=68)
    folder_entry.grid(row=0, column=1, padx=6, pady=4, sticky="we")

    def select_folder():
        folder = filedialog.askdirectory(title="Select a folder")
        if folder:
            folder_entry.delete(0, tk.END)
            folder_entry.insert(0, folder)

    tk.Button(frm, text="Browse…", command=select_folder).grid(row=0, column=2, padx=0, pady=4)

    # Keyword input
    tk.Label(frm, text="Keyword:").grid(row=1, column=0, sticky="w")
    keyword_entry = tk.Entry(frm, width=68)
    keyword_entry.grid(row=1, column=1, padx=6, pady=4, sticky="we")

    # Status line (no pop-ups)
    status_var = tk.StringVar(value="Ready.")
    tk.Label(frm, textvariable=status_var, anchor="w").grid(row=3, column=0, columnspan=3, sticky="we", pady=(10, 0))

    def on_run():
        folder = folder_entry.get().strip()
        kw = keyword_entry.get().strip()
        if not folder:
            messagebox.showerror("Error", "Please select a folder.")
            return
        if not kw:
            messagebox.showerror("Error", "Please enter a keyword.")
            return
        try:
            root.config(cursor="watch")
            root.update_idletasks()

            html = run_search(folder, kw)
            with open(out_path, "w", encoding="utf-8") as f:
                f.write(html)

            status_var.set(f"Results saved to: {out_path}")
            # Try to reuse an existing tab/window if possible
            webbrowser.open(f"file://{out_path}", new=0, autoraise=True)

        except FileNotFoundError as e:
            messagebox.showerror("Error", str(e))
        except Exception as e:
            messagebox.showerror("Error", f"An error occurred:\n{e}")
        finally:
            root.config(cursor="")

    tk.Button(frm, text="Search", command=on_run).grid(row=2, column=0, columnspan=3, pady=12)

    frm.columnconfigure(1, weight=1)
    root.mainloop()

# ========= CLI Mode (optional) =========
def parse_cli_args(argv):
    folder = None
    keyword = None
    i = 0
    while i < len(argv):
        a = argv[i]
        if a == "--folder" and i + 1 < len(argv):
            folder = argv[i+1]; i += 2
        elif a == "--keyword" and i + 1 < len(argv):
            keyword = argv[i+1]; i += 2
        else:
            i += 1
    return folder, keyword

def main():
    folder, keyword = parse_cli_args(sys.argv[1:])
    if folder and keyword:
        html = run_search(folder, keyword)
        out_path = os.path.abspath("pdf_search_results.html")
        with open(out_path, "w", encoding="utf-8") as f:
            f.write(html)
        print(f"Done. Open in browser: {out_path}")
    else:
        launch_gui()

if __name__ == "__main__":
    main()
