import uuid
import subprocess
import re
from pathlib import Path

from flask import Flask, render_template, request, jsonify, send_file

app = Flask(__name__)

BASE = Path(__file__).resolve().parent
DOWNLOADS = BASE / "downloads"

DOWNLOADS.mkdir(exist_ok=True)


def clean_name(name):
    name = str(name or "audio")

    name = re.sub(r'[\\/:*?"<>|]', "", name)
    name = re.sub(r"\s+", " ", name)
    name = name.strip(" .")

    if not name:
        name = "audio"

    return name[:180]


@app.route("/")
def index():
    return render_template("index.html")


@app.route("/api/search", methods=["POST"])
def api_search():

    data = request.get_json(silent=True) or {}

    query = str(data.get("query") or "").strip()

    if not query:
        return jsonify({
            "error": "Escribí una búsqueda"
        }), 400

    try:

        import yt_dlp

        opts = {
            "quiet": True,
            "no_warnings": True,
            "noplaylist": True,
            "extract_flat": True,

            "js_runtimes": {
                "node": None
            },

            "remote_components": {
                "ejs": ["github"]
            }
        }

        target = (
            query
            if query.startswith(("http://", "https://"))
            else "ytsearch8:" + query
        )

        with yt_dlp.YoutubeDL(opts) as ydl:

            info = ydl.extract_info(
                target,
                download=False
            )

        results = []

        if isinstance(info, dict):

            for item in info.get("entries") or []:

                if not isinstance(item, dict):
                    continue

                video_id = item.get("id")

                if not video_id:
                    continue

                results.append({
                    "id": video_id,
                    "title": item.get(
                        "title",
                        "Sin título"
                    ),
                    "url": (
                        item.get("webpage_url")
                        or
                        f"https://www.youtube.com/watch?v={video_id}"
                    )
                })

        return jsonify({
            "results": results
        })

    except Exception as e:

        print("SEARCH ERROR:", repr(e))

        return jsonify({
            "error": str(e)
        }), 500


@app.route("/api/download", methods=["POST"])
def api_download():

    data = request.get_json(silent=True) or {}

    url = str(data.get("url") or "").strip()

    fmt = str(
        data.get("format") or "mp3"
    ).lower()

    if not url:

        return jsonify({
            "error": "Falta la URL"
        }), 400

    if fmt not in ("mp3", "wav"):

        return jsonify({
            "error": "Formato inválido"
        }), 400

    token = uuid.uuid4().hex

    temp = DOWNLOADS / f"{token}.webm"

    try:

        # -------------------------------------------------
        # 1. OBTENER EL TÍTULO
        # -------------------------------------------------

        title_cmd = [
            "python",
            "-m",
            "yt_dlp",

            "--js-runtimes",
            "node",

            "--remote-components",
            "ejs:github",

            "--no-playlist",

            "--print",
            "%(title)s",

            "--skip-download",

            url
        ]

        title_result = subprocess.run(
            title_cmd,
            capture_output=True,
            text=True
        )

        if title_result.returncode != 0:

            raise RuntimeError(
                title_result.stderr.strip()
            )

        lines = [
            x.strip()
            for x in title_result.stdout.splitlines()
            if x.strip()
        ]

        title = (
            lines[-1]
            if lines
            else "audio"
        )

        title = clean_name(title)

        print("TÍTULO:", title)


        # -------------------------------------------------
        # 2. DESCARGAR
        # -------------------------------------------------

        download_cmd = [
            "python",
            "-m",
            "yt_dlp",

            "--js-runtimes",
            "node",

            "--remote-components",
            "ejs:github",

            "-f",
            "251",

            "-o",
            str(temp),

            url
        ]

        print("DESCARGANDO:", url)

        download_result = subprocess.run(
            download_cmd,
            capture_output=True,
            text=True
        )

        print(
            download_result.stdout
        )

        if download_result.returncode != 0:

            print(
                download_result.stderr
            )

            raise RuntimeError(
                download_result.stderr.strip()
            )

        if not temp.exists():

            raise RuntimeError(
                "yt-dlp no creó el archivo"
            )


        # -------------------------------------------------
        # 3. CREAR NOMBRE FINAL
        # -------------------------------------------------

        filename = f"{title}.{fmt}"

        final = DOWNLOADS / filename

        number = 2

        while final.exists():

            filename = (
                f"{title} ({number}).{fmt}"
            )

            final = DOWNLOADS / filename

            number += 1


        # -------------------------------------------------
        # 4. CONVERTIR
        # -------------------------------------------------

        if fmt == "mp3":

            ffmpeg_cmd = [
                "ffmpeg",
                "-y",

                "-i",
                str(temp),

                "-vn",

                "-codec:a",
                "libmp3lame",

                "-q:a",
                "2",

                str(final)
            ]

        else:

            ffmpeg_cmd = [
                "ffmpeg",
                "-y",

                "-i",
                str(temp),

                "-vn",

                "-codec:a",
                "pcm_s16le",

                str(final)
            ]


        print("CONVIRTIENDO:", fmt)

        ffmpeg_result = subprocess.run(
            ffmpeg_cmd,
            capture_output=True,
            text=True
        )

        if ffmpeg_result.returncode != 0:

            print(
                ffmpeg_result.stderr
            )

            raise RuntimeError(
                "FFmpeg no pudo convertir el archivo"
            )


        # Borrar temporal

        temp.unlink(
            missing_ok=True
        )


        print(
            "LISTO:",
            filename
        )


        return jsonify({
            "success": True,

            "filename": filename,

            "download": (
                f"/api/file/{token}"
            )
        })


    except Exception as e:

        print(
            "DOWNLOAD ERROR:",
            repr(e)
        )

        temp.unlink(
            missing_ok=True
        )

        return jsonify({
            "error": str(e)
        }), 500


@app.route("/api/file/<token>")
def api_file(token):

    # Buscar el archivo generado
    # recientemente.

    files = [
        p
        for p in DOWNLOADS.iterdir()
        if p.is_file()
        and p.suffix.lower()
        in (".mp3", ".wav")
    ]

    if not files:

        return jsonify({
            "error": "Archivo no encontrado"
        }), 404


    files.sort(
        key=lambda p: p.stat().st_mtime,
        reverse=True
    )

    path = files[0]


    return send_file(
        path,

        as_attachment=True,

        download_name=path.name
    )


if __name__ == "__main__":

    print()
    print("================================")
    print("            CUQUE")
    print("================================")
    print("Servidor:")
    print("http://127.0.0.1:5000")
    print("================================")
    print()

    app.run(
        host="0.0.0.0",
        port=5000,
        debug=False
    )
