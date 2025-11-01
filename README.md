# TESTE DE AI
estou tentando cria uma ia ou ao menos saber se e posivel criar uma onde ela aprende com o usuario e tambem tem um altonomi ONDE ELA MESMO PODE SAIR E BUSCAR CONHECIMENTO PARA PODER USAR
eu a chamei de Spirit, pois e o nome do personagem princiál de um dos meus filmes preferidos
a minha inteçao comk isso e trazer liberdade. Eu utilizo as AI que temos disponel hoje em dia mas sinto ela limitadas ao conhecimento e ao seu usuario por isso o nome Spirit que siguinifica espirito 

#ESPÍRITO - Assistente conversacional que aprende com o usuário e com a web 

# spirit.py
Usage:
  - Interactive chat mode: python spirit.py chat
  - Autonomous search: python spirit.py search
  - Export memory: python spirit.py export <path>
  - Import memory: python spirit.py import <path>
  - Show memory keys: python spirit.py list
"""
import sys
from ai_memoria import AIMemoria
from tools.autonomous_search import buscar_e_aprender

def chat_mode(ai: AIMemoria):
    print("SPIRIT interactive chat. Type 'exit' to quit, ':learn <question>|<answer>' to teach, ':del <question>' to delete.")
    while True:
        try:
            q = input("Você: ").strip()
        except (KeyboardInterrupt, EOFError):
            print("\nEncerrando SPIRIT.")
            break
        if not q:
            continue
        if q.lower() in ("exit", "quit"):
            break
        if q.startswith(":learn "):
            payload = q[len(":learn "):]
            if "|" in payload:
                pergunta, resposta = payload.split("|", 1)
                ai.ensinar(pergunta.strip(), resposta.strip())
            else:
                print("Formato: :learn pergunta|resposta")
            continue
        if q.startswith(":del "):
            pergunta = q[len(":del "):].strip()
            ok = ai.apagar(pergunta)
            print("Removido." if ok else "Não encontrado.")
            continue
        resp = ai.conversar(q, ask_if_unknown=True)
        if resp is None:
            print("[SPIRIT] Sem resposta.")
        else:
            print("SPIRIT:", resp)

def search_mode(ai: AIMemoria):
    q = input("Digite a query para pesquisa autônoma: ").strip()
    if not q:
        print("Query vazia.")
        return
    buscar_e_aprender(q, ai, max_sites=3, headless=True)
    print("Pesquisa autônoma concluída.")

def main():
    ai = AIMemoria(banco="spirit_memoria.json", auto_aprender=False, confirmar_salvamento=False)
    if len(sys.argv) < 2:
        print(__doc__)
        return
    cmd = sys.argv[1].lower()
    if cmd == "chat":
        chat_mode(ai)
    elif cmd == "search":
        search_mode(ai)
    elif cmd == "export" and len(sys.argv) >= 3:
        path = sys.argv[2]
        ai.exportar(path)
        print(f"Exportado para {path}")
    elif cmd == "import" and len(sys.argv) >= 3:
        path = sys.argv[2]
        ai.importar(path, override=False)
        print(f"Importado de {path}")
    elif cmd == "list":
        keys = list(ai.memoria.keys())
        print("Memória (chaves):")
        for k in keys:
            print("-", k)
    else:
        print("Comando desconhecido.")
        print(__doc__)

if __name__ == "__main__":
    main()
# ai_memoria.py
import os
import json
import threading
import re
from datetime import datetime
from typing import Callable, Optional, Dict, Any

SENSITIVE_PATTERNS = [
    re.compile(r"\b\d{11,16}\b"),
    re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"),
    re.compile(r"\b\d{3}\.?\d{3}\.?\d{3}-?\d{2}\b"),
]

def contains_sensitive(text: str) -> bool:
    if not text:
        return False
    for p in SENSITIVE_PATTERNS:
        if p.search(text):
            return True
    return False

class AIMemoria:
    def __init__(
        self,
        banco: str = "spirit_memoria.json",
        ask_callback: Optional[Callable[[str], str]] = None,
        auto_aprender: bool = True,
        confirmar_salvamento: bool = False,
        max_entries: int = 20000,
    ):
        self.banco = banco
        self._lock = threading.RLock()
        self.ask_callback = ask_callback or (lambda q: input(q + "\n> "))
        self.auto_aprender = bool(auto_aprender)
        self.confirmar_salvamento = bool(confirmar_salvamento)
        self.max_entries = int(max_entries)
        self._load_time = None
        self.memoria: Dict[str, Any] = self._carregar_memoria()

    # restante do código é igual, apenas prefira usar prefixos de log "SPIRIT"
    # por exemplo, em mensagens use "[SPIRIT]" em vez de "[AIMemoria]" ou "[SPIRYT]"

    # tools/autonomous_search.py
import time
import urllib.parse
from bs4 import BeautifulSoup
from ai_memoria import AIMemoria
from playwright.sync_api import sync_playwright
import requests

def extrair_texto_html(html: str) -> str:
    soup = BeautifulSoup(html, "lxml")
    article = soup.find("article")
    if article:
        return article.get_text(separator="\n").strip()
    paragraphs = soup.find_all("p")
    texts = [p.get_text().strip() for p in paragraphs if p.get_text().strip()]
    return "\n".join(texts[:8]).strip()

def buscar_links_bing(query: str, max_results=5):
    q = urllib.parse.quote_plus(query)
    url = f"https://www.bing.com/search?q={q}"
    headers = {"User-Agent": "SPIRIT/1.0 (+https://example.com)"}
    r = requests.get(url, headers=headers, timeout=15)
    r.raise_for_status()
    soup = BeautifulSoup(r.text, "lxml")
    links = []
    for li in soup.select("li.b_algo h2 a"):
        href = li.get("href")
        if href:
            links.append(href)
        if len(links) >= max_results:
            break
    return links

def sintetizar_texto(texto: str, max_chars=800) -> str:
    if not texto:
        return ""
    texto = texto.strip()
    primeiro = texto.split("\n\n")[0]
    return primeiro[:max_chars].strip()

def buscar_e_aprender(query: str, ai: AIMemoria, max_sites=3, headless=True):
    print(f"[SPIRIT][Autonomous] Buscando: {query}")
    links = buscar_links_bing(query, max_results=max_sites*3)
    if not links:
        print("[SPIRIT][Autonomous] Nenhum link encontrado.")
        return
    with sync_playwright() as pw:
        browser = pw.chromium.launch(headless=headless)
        page = browser.new_page(user_agent="SPIRIT/1.0 (+https://example.com)")
        visited = 0
        for href in links:
            if visited >= max_sites:
                break
            try:
                print(f"[SPIRIT][Autonomous] Visitando {href}")
                page.goto(href, timeout=30000)
                time.sleep(1.0)
                html = page.content()
                texto = extrair_texto_html(html)
                if not texto:
                    print("[SPIRIT][Autonomous] Conteúdo vazio, pulando.")
                    continue
                resposta = sintetizar_texto(texto)
                try:
                    ai.ensinar(query, resposta, salvar=True)
                    print(f"[SPIRIT][Autonomous] Aprendido de {href}")
                except Exception as e:
                    print(f"[SPIRIT][Autonomous] Não salvou: {e}")
                visited += 1
            except Exception as e:
                print(f"[SPIRIT][Autonomous] Erro ao visitar {href}: {e}")
        browser.close()

        
