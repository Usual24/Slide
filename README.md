# ======================================================================
#   TERMUX / PROOT UBUNTU 환경용
#   FLASK + Gemini API + HTML 저장 전용 (iframe 포함, 최대 30장)
# ======================================================================

import os
import json
import requests
from flask import Flask, request, redirect, render_template_string, send_file

# ----------------------------------------------------------------------
#                         기본 경로 설정
# ----------------------------------------------------------------------

ROOT = os.path.dirname(os.path.abspath(__file__))
SLIDES_DIR = os.path.join(ROOT, "slides")
DATA_FILE = os.path.join(SLIDES_DIR, "slide_data.json")

os.makedirs(SLIDES_DIR, exist_ok=True)

API_KEY = os.environ.get("GEMINI_API_KEY", "")
if not API_KEY:
    print("⚠️  GEMINI_API_KEY 환경변수 없음 — 슬라이드 생성 불가")

app = Flask(__name__)

GEMINI_URL = (
    "https://generativelanguage.googleapis.com/v1beta/models/"
    "gemini-2.0-flash:generateContent?key=" + API_KEY
)

# ----------------------------------------------------------------------
#                         유틸 함수
# ----------------------------------------------------------------------

def load_slide_data():
    if os.path.exists(DATA_FILE):
        try:
            return json.load(open(DATA_FILE, "r", encoding="utf-8"))
        except:
            return []
    return []

def save_slide_data(data):
    json.dump(data, open(DATA_FILE, "w", encoding="utf-8"), ensure_ascii=False, indent=4)

# ----------------------------------------------------------------------
#                   Gemini 슬라이드 생성 (HTML 저장 전용)
# ----------------------------------------------------------------------

def generate_slides(prompt: str, slide_count: int):
    if not API_KEY:
        return "API KEY 없음", []

    system_prompt = (
        "당신은 전문적인 프레젠테이션 제작기. "
        f"슬라이드 개수는 정확히 {slide_count}개. "
        "각 슬라이드는 JSON 배열 요소이며 slide_id, title, html_content 포함. "
        "html_content는 Tailwind CSS 포함 완전한 단일 HTML 문서로 생성."
    )

    payload = {
        "contents": [
            {
                "role": "user",
                "parts": [{"text": system_prompt + "\n\n주제: " + prompt}]
            }
        ],
        "generationConfig": {
            "responseMimeType": "application/json"
        }
    }

    try:
        res = requests.post(GEMINI_URL, json=payload)
        raw = res.json()
        text_output = raw["candidates"][0]["content"]["parts"][0]["text"]
        slides = json.loads(text_output)
    except Exception as e:
        return f"모델 오류: {e}", []

    new_data = []

    for idx, slide in enumerate(slides):
        sid = idx + 1
        html_path = os.path.join(SLIDES_DIR, f"{sid:02}.html")
        open(html_path, "w", encoding="utf-8").write(slide["html_content"])

        slide["id"] = sid
        slide["html_path"] = html_path
        new_data.append(slide)

    save_slide_data(new_data)
    return "생성 완료", new_data

# ----------------------------------------------------------------------
#                     Flask 라우팅
# ----------------------------------------------------------------------

@app.route("/")
def index():
    data = load_slide_data()
    if not data:
        html = """
        <h1>슬라이드 없음</h1>
        <p>아직 슬라이드가 없습니다. 아래에서 생성하세요.</p>
        <form method="POST" action="/generate">
            <input name="prompt" placeholder="주제" required>
            <input name="slide_count" type="number" value="5" min="1" max="30"> <!-- 최대 30장 -->
            <button type="submit">생성</button>
        </form>
        """
        return html

    htmls = [{"id": s["id"], "title": s["title"]} for s in data]

    html = """
    <h1>Slide 목록 (HTML 전용)</h1>
    <form method="POST" action="/generate">
        <input name="prompt" placeholder="주제" required>
        <input name="slide_count" type="number" value="5" min="1" max="30"> <!-- 최대 30장 -->
        <button type="submit">생성</button>
    </form>
    <hr>
    {% for s in slides %}
        <div style="margin-bottom:30px;">
            <h3>{{ s.title }}</h3>
            <iframe src="/file/{{ s.id }}" width="800" height="450" style="border:1px solid #ccc;"></iframe>
            <br>
            <a href="/edit/{{ s.id }}">편집</a>
        </div>
        <hr>
    {% endfor %}
    """
    return render_template_string(html, slides=htmls)

@app.route("/file/<int:sid>")
def serve_static(sid):
    data = load_slide_data()
    slide = next((x for x in data if x["id"] == sid), None)
    if not slide:
        return f"파일 없음: {sid}", 404
    return send_file(slide["html_path"])

@app.route("/generate", methods=["POST"])
def handle_generate():
    prompt = request.form.get("prompt")
    count = int(request.form.get("slide_count", "5"))
    if count > 30:
        count = 30
    if count < 1:
        count = 1

    # 기존 파일 삭제
    for f in os.listdir(SLIDES_DIR):
        os.remove(os.path.join(SLIDES_DIR, f))

    generate_slides(prompt, count)
    return redirect("/")

@app.route("/edit/<int:sid>")
def edit(sid):
    data = load_slide_data()
    slide = next((x for x in data if x["id"] == sid), None)
    if not slide:
        return redirect("/")

    html = f"""
    <h1>Slide {sid} 편집</h1>
    <form method="POST" action="/save/{sid}">
        제목: <input name="title" value="{slide['title']}"><br><br>
        <textarea name="html_content" rows="25" cols="100">{slide['html_content']}</textarea><br>
        <button type="submit">저장</button>
    </form>
    """
    return html

@app.route("/save/<int:sid>", methods=["POST"])
def save(sid):
    data = load_slide_data()
    slide = next((x for x in data if x["id"] == sid), None)
    if not slide:
        return redirect("/")

    slide["title"] = request.form.get("title")
    slide["html_content"] = request.form.get("html_content")
    save_slide_data(data)

    html_path = slide["html_path"]
    open(html_path, "w", encoding="utf-8").write(slide["html_content"])
    return redirect("/")

# ----------------------------------------------------------------------

if __name__ == "__main__":
    app.run(port=8080)
