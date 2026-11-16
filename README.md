main.py
Here is a complete, production-ready FastAPI backend (main.py) for inclusive-legal-os. It acts as a zero-trust local coordinator—uniting your Ollama local COGA text simplifier, Piper neural text-to-speech bridge, and GPG-signed SQLite accommodation ledger into a single asynchronous API.
FastAPI Backend (main.py)
Save this script inside your repository root or under a backend/ directory:
"""
Inclusive Legal OS - Unified Zero-Trust FastAPI Backend
License: AGPL-3.0-only
"""
import json
import subprocess
import hashlib
import sqlite3
from datetime import datetime, timezone
from pathlib import Path
import urllib.request

from fastapi import FastAPI, HTTPException, BackgroundTasks
from fastapi.responses import FileResponse
from pydantic import BaseModel, Field

app = FastAPI(
    title="Inclusive Legal OS API",
    description="Local zero-trust backend uniting W3C COGA simplification, Piper TTS, and GPG-signed accommodation logs.",
    version="1.0.0"
)

# Configuration paths & local endpoints
OLLAMA_ENDPOINT = "http://127.0.0.1:11434/api/generate"
DEFAULT_MODEL = "llama3.2:3b"
PIPER_BIN = "/usr/local/bin/piper"
PIPER_MODEL = "/opt/inclusive-legal-os/models/en_US-lessac-medium.onnx"
DB_PATH = Path.home() / ".local/share/inclusive-legal-os/accommodation_ledger.db"

COGA_SYSTEM_PROMPT = """You are an accessibility assistant adhering strictly to W3C COGA (Cognitive and Learning Disabilities Accessibility Task Force) Easy-Read guidelines.
Rewrite the user's input text following these mandatory rules:
1. Use short, direct sentences (maximum 12-15 words per sentence).
2. Use active voice only ("You must sign this form", NOT "This form must be signed").
3. One main idea per bullet point.
4. Define any unavoidable legal, medical, or technical word immediately in parentheses.
5. Never use idioms, metaphors, double negatives, or Latin legalese.
6. Highlight deadlines, money owed, or rights waived at the very top under 'IMPORTANT THINGS TO KNOW'.
Output valid JSON with keys: "important_warnings" (list of strings), "easy_read_steps" (list of strings), "defined_terms" (dict)."""

# Pydantic Schemas
class SimplifyRequest(BaseModel):
    text: str = Field(..., description="Raw text, EULA, or legal document to simplify.")
    model: str = Field(DEFAULT_MODEL, description="Ollama local model identifier.")

class TTSRequest(BaseModel):
    text: str = Field(..., description="Easy-Read text to convert to neural speech.")
    length_scale: float = Field(1.15, description="Pacing speed multiplier (1.15 = 15% slower).")

class LedgerLogRequest(BaseModel):
    module_name: str = Field(..., description="Name of the assistive module (e.g., 'cognitive-simplifier').")
    accommodation_type: str = Field(..., description="Type of accommodation used (e.g., 'easy-read-tts').")
    session_duration_sec: int = Field(..., description="Duration of the session in seconds.")
    gpg_key_id: str | None = Field(None, description="Optional GPG signing key ID for tamper-evident logging.")

def init_db():
    DB_PATH.parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(DB_PATH)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS accommodation_events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp_utc TEXT NOT NULL,
            module_name TEXT NOT NULL,
            accommodation_type TEXT NOT NULL,
            session_duration_sec INTEGER NOT NULL,
            record_sha256 TEXT NOT NULL,
            gpg_signature TEXT
        )
    """)
    conn.commit()
    conn.close()

@app.on_event("startup")
def startup_event():
    init_db()

@app.post("/api/simplify")
def simplify_endpoint(req: SimplifyRequest):
    payload = {
        "model": req.model,
        "system": COGA_SYSTEM_PROMPT,
        "prompt": f"Rewrite this text into Easy-Read JSON:\n\n{req.text}",
        "format": "json",
        "stream": False,
        "options": {"temperature": 0.1}
    }
    try:
        http_req = urllib.request.Request(
            OLLAMA_ENDPOINT,
            data=json.dumps(payload).encode("utf-8"),
            headers={"Content-Type": "application/json"},
            method="POST"
        )
        with urllib.request.urlopen(http_req, timeout=60) as resp:
            body = json.loads(resp.read().decode("utf-8"))
            parsed = json.loads(body["response"])
            return parsed
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Ollama local inference failed: {str(e)}")

@app.post("/api/tts")
def tts_endpoint(req: TTSRequest):
    output_wav = Path(f"/tmp/coga_tts_{datetime.now().timestamp()}.wav")
    cmd = [
        PIPER_BIN,
        "--model", PIPER_MODEL,
        "--length_scale", str(req.length_scale),
        "--output_file", str(output_wav)
    ]
    try:
        subprocess.run(cmd, input=req.text.encode("utf-8"), check=True, capture_output=True)
        return FileResponse(path=output_wav, media_type="audio/wav", filename="easy_read_speech.wav")
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Piper TTS synthesis failed: {str(e)}")

@app.post("/api/ledger/log")
def ledger_log_endpoint(req: LedgerLogRequest):
    ts = datetime.now(timezone.utc).isoformat()
    payload = {
        "timestamp_utc": ts,
        "module_name": req.module_name,
        "accommodation_type": req.accommodation_type,
        "session_duration_sec": req.session_duration_sec
    }
    canonical = json.dumps(payload, sort_keys=True).encode("utf-8")
    digest = hashlib.sha256(canonical).hexdigest()

    sig = None
    if req.gpg_key_id:
        cmd = ["gpg", "--batch", "--yes", "--armor", "--detach-sign", "--local-user", req.gpg_key_id]
        proc = subprocess.run(cmd, input=canonical, capture_output=True)
        if proc.returncode == 0:
            sig = proc.stdout.decode("utf-8")
        else:
            raise HTTPException(status_code=400, detail=f"GPG signing failed: {proc.stderr.decode('utf-8')}")

    conn = sqlite3.connect(DB_PATH)
    conn.execute(
        "INSERT INTO accommodation_events (timestamp_utc, module_name, accommodation_type, session_duration_sec, record_sha256, gpg_signature) VALUES (?, ?, ?, ?, ?, ?)",
        (ts, req.module_name, req.accommodation_type, req.session_duration_sec, digest, sig)
    )
    conn.commit()
    conn.close()

    return {
        "status": "success",
        "timestamp_utc": ts,
        "record_sha256": digest,
        "signed": bool(sig)
    }

Requirements (requirements.txt)
fastapi>=0.110.0
uvicorn>=0.28.0
pydantic>=2.6.0
requests>=2.31.0

How to Run Locally
 * Install dependencies:
   pip install -r requirements.txt

 * Start the FastAPI server using Uvicorn:
   uvicorn main:app --host 127.0.0.1 --port 8000 --reload

 * Access the interactive Swagger UI documentation at [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs) to test endpoints like /api/simplify, /api/tts, and /api/ledger/log directly.

# Quarkslab Publications

- [Quarkslab Publications](#quarkslab-publications)
  - [Conference Papers / Presentations](#conference-papers--presentations)
    - [2023](#2023)
    - [2022](#2022)
    - [2021](#2021)
    - [2020](#2020)
    - [2019](#2019)
    - [2018](#2018)
    - [2017](#2017)
    - [2016](#2016)
    - [2015](#2015)
    - [2013](#2013)
    - [2012](#2012)
  - [Hardware CTFs](#hardware-ctfs)

## Conference Papers / Presentations

### 2023

- [Google Apps Script 🖥️](confs/GreHack23/GreHack2023_Google_Apps_Script-v1.0.pptx)[📽️](https://www.youtube.com/watch?v=mHJ1xjH-pq0) *GreHack'23*
- [Dissecting the Modern Android Data Encryption Scheme 🖥️](Confs/Hardweario-NL-2023/HWIO23-Android-FBE-mrossibellom-dmelotti.pdf)[📽️](https://www.youtube.com/watch?v=I7_8OMcWPHg) *Hardwear.io NL'23*
- [Breaking Secure Boot on the Silicon Labs Gecko platform 🖥️](Confs/Hardweario-NL-2023/2023-11-03-hwio_nl-gecko_bootloader-madsquirrel.pdf)[📽️](https://www.youtube.com/watch?v=WgXw6GgYz78) *Hardwear.io NL'23*
- [Breaking Secure Boot on the Silicon Labs Gecko platform 🖥️](Confs/Ekoparty2023/2023-11-03-ekoparty-gecko_bootloader-fenrisfulsur.pdf)[📽️](https://www.youtube.com/watch?v=bWO9pIWd_Ok) *Ekoparty 2023*
- [On the All UR are to be considered harmful for fun and profit is the new cool trick, hackers hate it. Redux. 🖥️](Confs/Ekoparty2023/2023-11-01-ekoparty-keynote-iarce.pdf)[📽️](https://www.youtube.com/watch?v=YIcW7hnhBBE) *Ekoparty 2023*
- [Intel SGX assessment methodology 🖥️](Confs/AzureConfidentialComputing/AzureConfidentialComputing-IntelSGX-BlindAI-audit.pdf) *Azure Confidential Computing 2023*
- [Pyrrha: navigate easily into your system binaries 🖥️](Confs/HackLu23/pyrrha.pdf)[📽️](https://www.youtube.com/watch?v=-dMl-SvQl4k) *Hack.lu'23*
- [Fuzzing ntop 🖥️](Confs/ntopconf-2023/ntopconf-2023-fuzzing-ntop-rmori.pdf) *ntopconf'23*
- [Emulation de périphérique USB-ETH pour l’audit IoT/Automotive 🖥️](Confs/Barbhack23/Barbhack_2023_QB_Emulation_USB-ETH.pdf) *BarbHack'23*
- [Introduction au CarHacking Comment construire sa “Car-in-a-box” 🖥️](Confs/Barbhack23/Barbhack_2023_Workshop_Car_in_a_Box.pdf) *BarbHack'23*
- [Map your Firmware! 🖥️](Confs/PTS23/PTS2023-Talk-14-Pyrrha-map-your-firmware.pdf)[📽️](https://passthesalt.ubicast.tv/videos/2023-map-your-firmware/) *PTS'23*
- [For Science! - Using an Unimpressive Bug in EDK II To Do Some Fun Exploitation 🖥️](Confs/PTS23/PTS2023-Talk-03-EDK2.pdf)[📽️](https://passthesalt.ubicast.tv/videos/2023-for-science-using-an-unimpressive-bug-in-edk-ii-to-do-some-fun-exploitation/) *PTS'23*
- [Vulnerabilities in the TPM 2.0 reference implementation code 🖥️](Confs/PTS23/PTS2023-Talk-01-Vuln-in-TPM2-reference-implementation.pdf)[📽️](https://passthesalt.ubicast.tv/videos/2023-vulnerabilities-in-the-tpm-20-reference-implementation-code/) *PTS'23*
- [Parasitizing servers for fun and profit 🖥️](Confs/LeHack23/cauquil-Parasitizing_servers_for_fun_and_profit.pdf)[📽️](https://www.youtube.com/watch?v=3MXz4e4Qat0) *Le HACK'23*
- [Vulnerabilities in the TPM 2.0 Reference Implementation Code 🖥️](Confs/Troopers23/Troopers-Falcon-VulnerabilitiesintheTPMReferenceImplementationCode.pdf)[📽️](https://www.youtube.com/watch?v=GwK7YRT2V70) *Troopers'23*
- [Google Apps Script 🖥️](Confs/ESE23/ESE_Google_Apps_Script-v1.0.pdf) *ESE'23*
- [Who evaluates the evaluators ? 🖥️](Confs/WRACH2023/WRACH2023-who-evaluates-evaluators-slides-mvideau.pdf) *WRACH'23*
- [Dissecting the Modern Android Data Encryption Scheme 🖥️](Confs/Recon23/Recon23-Android-FBE-mrossibellom-dmelotti.pdf)[📽️](https://www.youtube.com/watch?v=unRFQhkK1DE) *Recon'23*
- [Trace-based approach to compiler debugging 🖥️](Confs/GDRGPL23/pres.pdf) *GDR GPL National Days'23*
- [Exploring OpenSSL Engines to Smash Cryptography 🖥️](Confs/SSTIC23/SSTIC2023-Slides-exploring_openssl_engines_to_smash_cryptography-goudarzi_valadon.pdf)[📜](Confs/SSTIC23/SSTIC2023-Article-exploring_openssl_engines_to_smash_cryptography-goudarzi_valadon.pdf)[📽️](https://static.sstic.org/videos2023/1080p/exploring_openssl_engines_to_smash_cryptography.mp4) *SSTIC'23* 
- [peetch: an eBPF based Networking Tool 🖥️](Confs/SSTIC23/SSTIC2023-Slides-peetch-valadon.pdf)[📽️](https://static.sstic.org/videos2023/1080p/peetch.mp4) *SSTIC'23* 
- [Rétro-ingénierie et détournement de piles protocolaires embarquées 🖥️](Confs/SSTIC23/SSTIC2023-Slides-reverse_de_piles_protocolaires_embarquees_esp32-cauquil_cayre.pdf)[📜](Confs/SSTIC23/SSTIC2023-Article-reverse_de_piles_protocolaires_embarquees_esp32-cauquil_cayre.pdf)[📽️](https://static.sstic.org/videos2023/1080p/reverse_de_piles_protocolaires_embarquees_esp32.mp4) *SSTIC'23* 
- [ESPwn32: Hacking with ESP32 System-on-Chips 🖥️](Confs/WOOT-2023/22-ESPwn32_Hacking_with_ESP32_SoCs.pdf)[📜](Confs/WOOT-2023/woot23-paper22.pdf) *WOOT'23* 
- [Emulating RH850 for fun and vulnerability research 🖥️](Confs/QPSS2023/QPSS2023-Emulating_Renesas_RH850-dcauquil-slides.pdf) *QPSS2023*
- [PASTIS - A Collaborative Approach to Combine Heterogeneous Software Testing Techniques 🖥️](Confs/SBTF-ICSE-2023/SBFT2023-PASTIS-rdavid.pdf) [📜](Confs/SBTF-ICSE-2023/SBFT2023-PASTIS-paper-rdavid.pdf) *SBTF2023*
- [For Science! - Using an Unimpressive Bug in EDK II To Do Some Fun Exploitation 🖥️](Confs/Sthack23/uefi_tcg2smm.pdf)[📽️](https://www.youtube.com/watch?v=SjFTqqiHVBE) *StHack'23*
- [Trying to break randomness with statistics in less than 5minutes 🖥️](Confs/Sthack23/brocas_forgette_pwn2own_sthack23.pdf) *StHack'23*
- [Reflections on Supply chain security 🖥️](Confs/CERT2023/20230424-CERTvendors-conf-Reflections_on_Supply-chain_security-iarce.pdf) *CERT Vendor Conference'23*
- [Weaponizing ESP32 RF Stacks 🖥️](Confs/THCon23/THCon23-Cayre-Cauquil-Weaponizing-ESP32-RF-stacks.pdf)[📽️](https://www.youtube.com/live/Mks2pZ4Subw?feature=share&t=13005) *THCon'23*
- [Whatever Pown2own 🖥️](Confs/INS23/Whatever_Pown2own.pdf) *Insomni'hack'23*
- [Traceability of the compilation process 🖥️](Confs/CLAP-HiFi-LVP-2023/pres-clap.pdf) *CLAP-HiFi-LVP'23*


### 2022

- [Attack on Titan M 🖥️](Confs/Troopers22/Melotti-RossiBellom_TitanM.pdf)[📽️](https://www.youtube.com/watch?v=clhzUGcx47A) *Troopers'22*
- [Attack on Titan M, Reloaded 🖥️](Confs/BlackHat-USA-22/US-22-Melotti-Attack-on-Titan-M-Reloaded.pdf)[📽️](https://www.youtube.com/watch?v=bCjkAkXiwK4) *BlackHatUS'22*
- [Attack on Titan M, Reloaded 🖥️](Confs/Ekoparty2022/Ekoparty2022%20Attack%20on%20Titan%20M%20Reloaded%20-%20Melotti.pdf)[📽️](https://www.youtube.com/watch?v=AKTjhjrUEKw) *Ekoparty'22*
- [kdigger - A Context Discovery Tool for Kubernetes Penetration Testing 🖥️](Confs/BlackHat-Asia-22/blackhat-asia-arsenal-kdigger.pdf) *BlackHatAsia'22*
- [Building a Commit-level Dataset of Real-World Vulnerabilities 🖥️](Confs/CODASPY-22/2022-codaspy-achallade-BuildingACommitLevelDataset.pdf)[📽️](https://www.youtube.com/watch?v=b_T4kCJ-P3o) *CODASPY'22*
- [When eBPF meets TLS 🖥️](Confs/CanSecWest-2022/When%20eBPF%20meets%20TLS.pdf) *CanSecWest'22*
- [Quokka - A Fast and Accurate Binary Exporter 🖥️](Confs/GreHack2022/Quokka-achallande-2022.pdf)[📽️](https://www.youtube.com/watch?v=3kbAW1LbsAo) *GreHack'22*
- [A journey of fuzzing Nvidia graphic driver leading to LPE exploitation 🖥️](Confs/Hexacon-2022/fuzzing_NVIDIA_drivers-tdore.pdf)[📽️](https://www.youtube.com/watch?v=ffBuY7ON57g) *Hexacon'22*
- [Symbolic Execution the Swiss-Knife of the Reverse Engineering Toolbox 🖥️](Confs/KLEE-Workshop-2022/2022-09-klee-workshop-rdavid.pdf)[📽️](https://www.youtube.com/watch?v=PNbNtTa5Sp4) *KleeWorkshop'22*
- [From Offensive to Defensive Security 🖥️](Confs/Les%20Assises%20-%202022/From%20Offensive%20to%20Defensive%20Security.pdf) *Les Assises'22*
- [TPM is not the holy way 🖥️](Confs/SSTIC22/SSTIC2022-Slides-tpm_is_not_the_holy_way-forgette.pdf)[📽️](https://static.sstic.org/videos2022/1080p/tpm_is_not_the_holy_way.mp4)[📜](Confs/SSTIC22/SSTIC2022-Article-tpm_is_not_the_holy_way-forgette.pdf) *SSTIC'22*
- [Binbloom v2 - Ceci est une (r)evolution 🖥️](Confs/SSTIC22/sstic22-binbloom-v2.pdf)[📽️](https://static.sstic.org/videos2022/1080p/binbloom_v2.mp4) *SSTIC'22*
- [Mattermost End-to-End Encryption Plugin 🖥️](Confs/PTS22/PTS2022-Talk-01-Mattermost-e2ee-plugin.pdf)[📽️](https://archives.pass-the-salt.org/Pass%20the%20SALT/2022/videos/PTS2022-Talk-01-mattermost-end-to-end-encryption-plugin.mp4) *PTS'22*
- [Binbloom Reloaded 🖥️](Confs/PTS22/PTS2022-Talk-12-Binbloom-reloaded.pdf)[📽️](https://archives.pass-the-salt.org/Pass%20the%20SALT/2022/videos/PTS2022-Talk-12-binbloom-reloaded.mp4) *PTS'22*
- [kdigger - Kubernetes focused container assessment and context discovery tool for penetration testing 🖥️](Confs/PTS22/PTS2022-Talk-22-kdigger.pdf)[📽️](https://archives.pass-the-salt.org/Pass%20the%20SALT/2022/videos/PTS2022-Talk-22-kdigger-a-context-discovery-tool-for-kubernetes-penetration-testing.mp4) *PTS'22*

### 2021

- [From source code to crash test-cases through software testing automation 🖥️](Confs/C%26ESAR-2021/CESAR-2021_slides_2-2.pdf) *C&ESAR'22*
- [Wookey: Episode VII - The Force Awakens 🖥️](Confs/GreHack2021/GH2021%20Wookey:%20Episode%20VII%20-%20The%20Force%20Awakens%20-%20Teuwen.pdf)[📽️](https://www.youtube.com/watch?v=aAuiJZatYR8) *GreHack'21*
- [EEPROM - It will all End in Tears (EN) 🖥️](Confs/Hardweario-NL-2021/teuwen_herrmann_eeprom_tears_hwio_nl_2021_slides.pdf)[📽️](https://www.youtube.com/watch?v=zZp5h0Tdkhk) *Hardwear.io NL'21* 
- [EEPROM - It Will All End in Tears (FR) 🖥️](Confs/SSTIC21/SSTIC2021-Slides-eeprom_it_will_all_end_in_tears-herrmann_teuwen.pdf)[📽️](https://static.sstic.org/videos2021/1080p/vostfr-eeprom_it_will_all_end_in_tears.mp4)[📜](Confs/SSTIC21/SSTIC2021-Article-eeprom_it_will_all_end_in_tears-herrmann_teuwen.pdf) *SSTIC'21*
- [Meet Piotr, a firmware emulation tool for trainers and researchers 🖥️](Confs/PTS21/PTS2021-Talk-16-piotr.pdf)[📽️](https://archives.pass-the-salt.org/Pass%20the%20SALT/2021/videos/PTS2021-Talk-16-piotr.mp4) *PTS'21*
- [Reversing And Fuzzing The Google Titan M Chip 🖥️](Confs/ROOTS2021/DamianoMelotti_ReversingAndFuzzingTheGoogleTitanMChip_paper.pdf) *ROOTS'21*
- [QBDL - QuarkslaB Dynamic Loader 🖥️](Confs/SSTIC21/SSTIC-21-QBDL-aguinet-rthomas.pdf)[📽️](https://static.sstic.org/videos2021/1080p/qbdl_quarkslab_dynamic_loader.mp4) *SSTIC'21*
- [Mining AOSP Dependency Graph for Security 🖥️](Confs/SSTIC21/SSTIC2021-Slides-bgraph-challande_renault_david.pdf)[📽️](https://static.sstic.org/videos2021/1080p/bgraph.mp4)[📜](Confs/SSTIC21/SSTIC2021-Article-bgraph-challande_renault_david.pdf) *SSTIC'21*
- [2021: A Titan M Odyssey 🖥️](Confs/BlackHat-Europe-21/EU-21-Rossi_Bellom-2021_A_Titan_M_Odyssey.pdf)[📽️](https://www.youtube.com/watch?v=UNPblJup5ko)[📜](Confs/BlackHat-Europe-21/EU-21-Rossi_Bellom-2021_A_Titan_M_Odyssey-wp.pdf) *BlackHatEU'21*

### 2020

- [QSynth - A Program Synthesis approach for Binary Code Deobfuscation 🖥️](Confs/BAR20/bar_slides.pdf) *BAR'20*
- [Collision-Based Attacks Against Whiteboxes with QBDI 🖥️](Confs/Barbhack20/20-Barbhack-Collision-Based-Attacks-Against-Whiteboxes-with-QBDI.pdf) *BarbHack'20*
- [Dynamic Binary Instrumentation Techniques to Address Native Code Obfuscation 🖥️](Confs/BlackHat-Asia-20/asia-20-Thomas-Dynamic-Binary-Instrumentation-Techniques-to-Address-Native-Code-Obfuscation.pdf)[📽️](https://www.youtube.com/watch?v=MRku-2fW42w)[📜](Confs/BlackHat-Asia-20/asia-20-Thomas-Dynamic-Binary-Instrumentation-Techniques-to-Address-Native-Code-Obfuscation-wp.pdf) *BlackHat Asia'20*
- [Why are Frida and QBDI a Great Blend on Android? 🖥️](Confs/PTS20/PTS2020-Talk-05-Frida_QBDI.pdf)[📽️](https://archives.pass-the-salt.org/Pass%20the%20SALT/2020/videos/PTS2020-Talk-05-Frida_QBDI.mp4) *PTS'20*
- [Fuzz and Profit with WHVP 🖥️](Confs/SSTIC20/AumaitreDamien_FuzzAndProfitWithWHVP_slides.pdf)[📽️](https://static.sstic.org/videos2020/1080p/FuzzAndProfitWithWHVP.mp4)[📜](Confs/SSTIC20/AumaitreDamien_FuzzAndProfitWithWHVP.pdf) *SSTIC'20*

### 2019

- [Old New Things: An examination of the Philips TriMedia architecture 🖥️](Confs/Troopers19/trimedia-research-nriva.pdf)[📽️](https://www.youtube.com/watch?v=FPoSzLHa7dY) *Troopers'19*
- [Grey-box attacks, four years later 🖥️](Confs/WhibOx2019/WhibOx2019_greybox_attacks_4years_later_pteuwen.pdf) *WhibOx'19*

### 2018

- [QBDI - Implementing an LLVM based Dynamic Binary Instrumentation framework 🖥️](Confs/34C3-2018/QBDI_34c3-chubain-ctessier.pdf)[📽️](https://www.youtube.com/watch?v=Zt74lOuU6zc) *34C3'18*
- [Automatizing vulnerability research 🖥️](Confs/Cisco18/18-cisco-auto-vuln-research.pdf) *CISCO'18*
- [Symbolic deobfuscation: from virtualized code back to the original 🖥️](Confs/DIMVA18/DIMVA2018-deobfuscation-salwan-bardin-potet.pdf) *DIMVA'18*
- [Vulnerability Research - What It Takes to Keep Going and Going and Going 🖥️](Confs/HITB18/D1T2%20-%20Vulnerability%20Research%20-%20What%20It%20Takes%20to%20Keep%20Going%20and%20Going%20and%20Going%20-%20Cedric%20Tessier%20Fred%20Raynal.pdf) *HITB'18*
- [Static instrumentation based on executable file formats 🖥️](Confs/PTS18/PTS18-static-instrumentation-rthomas.pdf)[📽️](https://archives.pass-the-salt.org/Pass%20the%20SALT/2018/videos/PTS2018-03-static-instrumentation-based-on-executable-file-formats_2160p.mp4) *PTS'18*
- [Static instrumentation based on executable file formats 🖥️](Confs/Recon18/Recon18-Static-Instrumentation-rthomas.pdf) *Recon'18*
- [Attacking Serial Flash Chip: Case Study of a Black Box 📜](Confs/SSTIC18/SSTIC2018-Article-attacking_serial_flash_chip_case_study_of_a_black_box_device-benoit_heilles_teuwen.pdf)[📽️](https://static.sstic.org/videos2018/SSTIC_2018-06-13_P11.mp4) *SSTIC'18*

### 2017

- [LIEF: Library to Instrument Executable Formats 🖥️](Confs/RMLL17/17-RMLL-LIEF-rthomas.pdf)[📽️](https://archives.pass-the-salt.org/RMLL%20Security%20Tracks/2017/videos/RMLL-Sec-2017-lief_63569_720p.mp4) *RMLL'17*
- [Playing with Binary Analysis: Deobfuscation of VM based software protection / Desobfuscation binaire : Reconstruction de fonctions virtualisées 🖥️](Confs/SSTIC17/SSTIC2017_Deobfuscation_of_VM_based_software_protection.pdf)[📽️](https://static.sstic.org/videos2017/SSTIC_2017-06-07_P08.mp4)[📜](Confs/SSTIC17/SSTIC2017-Article-desobfuscation_binaire_reconstruction_de_fonctions_virtualisees-salwan_potet_bardin.pdf) *SSTIC'17*


### 2016

- [Design de cryptographie white-box : et a la fin, c'est Kerckhoffs qui gagne 🖥️](Confs/SSTIC16/SSTIC2016-Slides-design_de_cryptographie_white-box_et_a_la_fin_c_est_kerckhoffs_qui_gagne-hubain_teuwen_1.pdf)[📽️](http://static.sstic.org/videos2016/SSTIC_2016-06-03_P05.mp4)[📜](Confs/SSTIC16/SSTIC2016-Article-design_de_cryptographie_white-box_et_a_la_fin_c_est_kerckhoffs_qui_gagne-hubain_teuwen_1.pdf) *SSTIC'16*
- [Dynamic Binary Analysis and Obfuscated Codes 🖥️](Confs/StHack16/sthack2016-jsalwan-rthomas.pdf) *StHack'16*
- [How Triton can help to reverse virtual machine based software protections 🖥️](Confs/CSAW16/csaw2016-sos-jsalwan-rthomas.pdf) *CSAW'16*

### 2015

- [Some technical & scientific challenges I'd like to have working solutions for 🖥️](Confs/SAS15/15-09-sas.pdf) *SAS'15*
- [Triton: Concolic Execution Framework 🖥️](Confs/SSTIC15/SSTIC2015_English_slide_detailed_version_Triton_Concolic_Execution_FrameWork_FSaudel_JSalwan.pdf), [Fr](SSTIC15/SSTIC2015_French_Paper_Triton_Framework_dexecution_Concolique_FSaudel_JSalwan.pdf) *SSTIC'15*
- [Dynamic Binary Analysis and Instrumentation Covering a function using a DSE approach 🖥️](Confs/SecurityDay15/SecurityDay2015_dynamic_symbolic_execution_Jonathan_Salwan.pdf) *SecurityDay'15* \-\- [Dynamic Binary Analysis and Instrumentation Covering a function using a DSE approach 🖥️](Confs/StHack15/StHack2015_Dynamic_Behavior_Analysis_using_Binary_Instrumentation_Jonathan_Salwan.pdf) *StHack'15*

### 2013

- [UEFI and Dreamboot 🖥️](Confs/SSTIC13/D2T1-SebastienKaczmarek-Dreamboot_UEFI_Bootkit.pdf) *SSTIC'13*

### 2012

- [Pwn@Home: An Attack Path to jailbreaking your home router 🖥️](Confs/HITB12/12-10-HTIB-router.pdf) *HTIB'12*
- [SSTIC, les 10 ans 🖥️](Confs/SSTIC12/SSTIC-10ans-NF-FR-PB-FINAL.pdf) *SSTIC'12*

## Hardware CTFs

- First Edition
  - [Prize Ceremony, Hardwear.io NL 2017 🖥️](HardwareCTF/201709_v1_hardweario_nl.pdf)
- Second Edition
  - [Prize Ceremony, Hardwear.io NL 2018 🖥️](HardwareCTF/201809_v2_hardweario_nl.pdf)
  - [Prize Ceremony, Hardwear.io USA 2019 🖥️](HardwareCTF/201906_v2_hardweario_usa.pdf)
- Third Edition
  - [Prize Ceremony, Hardwear.io NL 2019 🖥️](HardwareCTF/201909_v3_hardweario_nl.pdf)
  - [Prize Ceremony, Nullcon Goa 2020 🖥️](HardwareCTF/202003_v3_nullcon.pdf)
- Fourth Edition
  - [Prize Ceremony, Hardwear.io NL 2021 🖥️](HardwareCTF/202110_v4_hardweario_nl.pdf)
  - [Prize Ceremony, Hardwear.io USA 2022 🖥️](HardwareCTF/202206_v4_hardweario_usa.pdf)
  - [Prize Ceremony, Nullcon Goa 2022 🖥️](HardwareCTF/202209_v4_nullcon.pdf)
- Fifth Edition
  - [Prize Ceremony, Hardwear.io NL 2022 🖥️](HardwareCTF/202210_v5_hardweario_nl.pdf)
  - [Prize Ceremony, Hardwear.io USA 2023 🖥️](HardwareCTF/202306_v5_hardweario_usa.pdf)
- Sixth Edition
  - [Prize Ceremony, Hardwear.io NL 2023 🖥️](HardwareCTF/202311_v6_hardweario_nl.pdf)
