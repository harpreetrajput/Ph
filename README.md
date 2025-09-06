# prediction_bot.py
import urllib.request, urllib.parse, json, time, os
from collections import Counter

# ---- BOT TOKEN (already added) ----
TOKEN = "8271047660:AAGN9GmDTVaa3XRfuSaz77oJomnsfRykdAU"
# ----------------------------------

API_URL = f"https://api.telegram.org/bot{TOKEN}/"
DATA_FILE = "numbers.json"
MAX_KEEP = 100

# Load saved numbers if exist
if os.path.exists(DATA_FILE):
    try:
        with open(DATA_FILE, "r") as f:
            numbers = json.load(f)
    except:
        numbers = []
else:
    numbers = []

def save_numbers():
    with open(DATA_FILE, "w") as f:
        json.dump(numbers, f)

def get_updates(offset=None, timeout=50):
    params = {}
    if offset is not None:
        params['offset'] = offset
    params['timeout'] = timeout
    url = API_URL + "getUpdates?" + urllib.parse.urlencode(params)
    with urllib.request.urlopen(url, timeout=timeout+10) as resp:
        return json.load(resp)

def send_message(chat_id, text):
    data = urllib.parse.urlencode({'chat_id': str(chat_id), 'text': text})
    req = urllib.request.Request(API_URL + "sendMessage", data=data.encode('utf-8'))
    with urllib.request.urlopen(req, timeout=10) as resp:
        return json.load(resp)

def compute_stats(nums):
    total = len(nums)
    freq = Counter(nums)
    odd = sum(v for k,v in freq.items() if k%2==1)
    even = total - odd
    big = sum(v for k,v in freq.items() if k>=5)
    small = total - big

    last_pos = {}
    for idx, val in enumerate(nums):
        last_pos[val] = idx
    per = []
    for d in range(10):
        f = freq.get(d,0)
        last = last_pos.get(d, None)
        missing = (total - 1 - last) if last is not None else total
        avg_missing = round(total / f,1) if f>0 else total
        maxc = 0
        cur = 0
        for v in nums:
            if v == d:
                cur += 1
                if cur > maxc: maxc = cur
            else:
                cur = 0
        score = (missing - avg_missing) + f*0.2 + maxc*0.5
        per.append((d, f, missing, avg_missing, maxc, round(score,2)))
    per_sorted = sorted(per, key=lambda x: x[5], reverse=True)
    return {
        'total': total, 'freq': freq, 'odd': odd, 'even': even,
        'big': big, 'small': small, 'per_sorted': per_sorted
    }

def analyze_and_reply(chat_id, num):
    numbers.append(num)
    if len(numbers) > MAX_KEEP:
        del numbers[:-MAX_KEEP]
    save_numbers()
    stats = compute_stats(numbers)
    top3 = stats['per_sorted'][:3]
    lines = []
    lines.append(f"✅ Saved: {num}")
    lines.append(f"📊 Last {stats['total']} rounds:")
    lines.append(f"Odd: {stats['odd']}  Even: {stats['even']}")
    lines.append(f"Big: {stats['big']}  Small: {stats['small']}")
    lines.append("Top candidates (digit - freq - score):")
    for d,freq,missing,avg,maxc,score in top3:
        lines.append(f"{d} - {freq} times - score {score}")
    suggestion = top3[0][0] if top3 else "—"
    lines.append(f"🔮 Suggested next: {suggestion}")
    send_message(chat_id, "\n".join(lines))

def handle_stats_command(chat_id):
    stats = compute_stats(numbers)
    top3 = stats['per_sorted'][:3]
    lines = []
    lines.append(f"📈 Stored numbers: {stats['total']}")
    lines.append(f"Odd: {stats['odd']}  Even: {stats['even']}")
    lines.append(f"Big: {stats['big']}  Small: {stats['small']}")
    lines.append("Top 3 digits (digit - freq):")
    for d,freq,_,_,_,_ in top3:
        lines.append(f"{d} - {freq}")
    send_message(chat_id, "\n".join(lines))

offset = None
print("Bot started. Waiting for messages... (Keep app open)")
while True:
    try:
        updates = get_updates(offset, timeout=60)
        for u in updates.get('result', []):
            offset = u['update_id'] + 1
            msg = u.get('message') or u.get('edited_message')
            if not msg:
                continue
            text = msg.get('text','').strip()
            chat_id = msg['chat']['id']
            if text == "/stats":
                handle_stats_command(chat_id)
                continue
            if text.isdigit():
                n = int(text)
                if 0 <= n <= 9:
                    analyze_and_reply(chat_id, n)
                else:
                    send_message(chat_id, "❌ Send single digit 0–9 only.")
            else:
                send_message(chat_id, "Send a single digit 0–9 to save & analyze, or /stats to view summary.")
    except Exception as ex:
        print("Error:", ex)
        time.sleep(5)
