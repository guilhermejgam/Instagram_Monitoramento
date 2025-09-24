import os
import json
import time
import requests
import hashlib
from datetime import datetime
from playwright.sync_api import sync_playwright

PROFILE_URL = "https://www.instagram.com/guilherme.martins.uk/"
STATE_FILE = r"C:\armazena\instagram_state.json" # Há necessidade da gravação de sessão
OUTPUT_DIR = r"C:\armazena\guilherme.martins.uk"
CHECK_INTERVAL = 30  

os.makedirs(OUTPUT_DIR, exist_ok=True)
STATE_JSON_FILE = os.path.join(OUTPUT_DIR, "last_state.json")

def load_last_state():
    if os.path.exists(STATE_JSON_FILE):
        with open(STATE_JSON_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {}


def save_state(state):
    with open(STATE_JSON_FILE, "w", encoding="utf-8") as f:
        json.dump(state, f, ensure_ascii=False, indent=4)


def image_url_changed(old_url, new_url, folder):
    if not old_url or not new_url:
        return True
        
    old_prefix = old_url.split("?")[0]
    new_prefix = new_url.split("?")[0]
    if old_prefix != new_prefix:
        return True
    try:
        resp = requests.get(new_url)
        if resp.status_code != 200:
            return False
        new_hash = hashlib.md5(resp.content).hexdigest()
        for f in os.listdir(folder):
            if f.endswith(".jpg"):
                with open(os.path.join(folder, f), "rb") as old_f:
                    if hashlib.md5(old_f.read()).hexdigest() == new_hash:
                        return False
        return True
    except:
        return True

def download_profile_img(url, folder, timestamp):
    if not url:
        return None
    filename = os.path.join(folder, f"profile_{timestamp}.jpg")
    try:
        resp = requests.get(url, stream=True)
        if resp.status_code == 200:
            with open(filename, "wb") as f:
                for chunk in resp.iter_content(1024):
                    f.write(chunk)
            return filename
    except Exception as e:
        print(f"Erro ao baixar a foto do perfil: {e}")
    return None


def extract_profile_data(page):

    page.goto(PROFILE_URL, timeout=60000, wait_until="domcontentloaded")
    page.wait_for_timeout(3000)  # espera JS carregar

    data = {}

    # Foto do perfil
    profile_img_elem = page.query_selector("header img")
    data["profile_img"] = profile_img_elem.get_attribute("src") if profile_img_elem else None

    #Bio
    meta_desc = page.query_selector("meta[name='description']")
    bio_text = meta_desc.get_attribute("content") if meta_desc else ""
    if bio_text:
        bio_text = bio_text.split("•")[0].strip()
    data["bio"] = bio_text

    #posts, seguidores, seguindo
    stats_elems = page.query_selector_all("header section ul li")
    data["posts"] = stats_elems[0].query_selector("span").get_attribute("title") or stats_elems[0].query_selector("span").inner_text() if len(stats_elems) >= 1 else ""
    data["followers"] = stats_elems[1].query_selector("span").get_attribute("title") or stats_elems[1].query_selector("span").inner_text() if len(stats_elems) >= 2 else ""
    data["following"] = stats_elems[2].query_selector("span").inner_text() if len(stats_elems) >= 3 else ""

    data["timestamp"] = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    return data


def compare_states(old, new):
    changes = {}
    for key in ["profile_img", "bio", "posts", "followers", "following"]:
        if new.get(key) != old.get(key):
            changes[key] = {"old": old.get(key), "new": new.get(key)}
    return changes


def print_changes_table(changes):
    if not changes:
        return
    print(f"\n[Alterações detectadas em {datetime.now().strftime('%H:%M:%S')}]")
    print(f"{'Campo':12} | {'Antes':40} | {'Agora':40}")
    print("-" * 100)
    for k, v in changes.items():
        old = v['old'] if v['old'] else ""
        new = v['new'] if v['new'] else ""
        campo_pt = {
            "profile_img": "Foto Perfil",
            "bio": "Bio",
            "posts": "Posts",
            "followers": "Seguidores",
            "following": "Seguindo"
        }.get(k, k)
        print(f"{campo_pt:12} | {old[:40]:40} | {new[:40]:40}")
    print("-" * 100)


def monitor_profile():
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        context = browser.new_context(storage_state=STATE_FILE,
                                      viewport={"width": 1280, "height": 800},
                                      user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
                                                 "(KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36")
        page = context.new_page()
        last_state = load_last_state()

        print(f"Monitorando {PROFILE_URL} a cada {CHECK_INTERVAL} segundos...")

        while True:
            try:
                page.reload(wait_until="domcontentloaded")
                page.wait_for_timeout(3000)

                current_state = extract_profile_data(page)
                changes = compare_states(last_state, current_state)

                # Tratamento especial da foto do perfil
                if "profile_img" in changes:
                    if image_url_changed(last_state.get("profile_img"), current_state["profile_img"], OUTPUT_DIR):
                        img_file = download_profile_img(current_state["profile_img"], OUTPUT_DIR, current_state["timestamp"])
                        if img_file:
                            print(f"Foto do perfil alterada e salva em: {img_file}")
                    else:
                        del changes["profile_img"]

                if changes:
                    print_changes_table(changes)
                    save_state(current_state)
                    last_state = current_state
                else:
                    print(f"[{datetime.now().strftime('%H:%M:%S')}] Nenhuma alteração detectada.")

            except Exception as e:
                print(f"Erro ao monitorar perfil: {e}")

            time.sleep(CHECK_INTERVAL)


if __name__ == "__main__":
    monitor_profile()
