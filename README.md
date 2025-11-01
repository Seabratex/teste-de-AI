# teste-de-AI
essa foi uma tentativa de criação de uma ai  onde ela vai  aprendendo com a pessoa que vai utilizando e se aprimorando com o tempo sozinha. Ainda vou melhorar mais ainda foi so um teste 
atualizçao 01:11:25
vamos para as alteraçoes feitas, pedir ajuda a AI do proprio git para mudar algumas coisas 

import os
import json
import threading
import re
from datetime import datetime
from typing import Callable, Optional, Dict, Any

SENSITIVE_PATTERNS = [
    # exemplos: números longos (possível cartão/CPF), emails, telefones simples — personalize conforme necessário
    re.compile(r"\b\d{11,16}\b"),  # 11-16 dígitos seguidos (telefone, cpf+zeros, cartão)
    re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"),  # email
    re.compile(r"\b\d{3}\.??\d{3}\.??\d{3}-?\d{2}\b"),  # cpf
]

def contains_sensitive(text: str) -> bool:
    if not text:
        return False
    for p in SENSITIVE_PATTERNS:
        if p.search(text):
            return True
    return False

class AIMemoria:
    """
    Memória simples persistente que aprende com o usuário.
    Principais recursos:
    - conversar(pergunta): responde se já sabe; se não sabe, pode perguntar ao usuário e aprender.
    - ensinar(pergunta, resposta): ensinar explicitamente.
    - apagar(pergunta): remover da memória.
    - exportar()/importar() para backup.
    - opções: auto_aprender (True/False), confirmar_salvamento, max_entries.
    """

    def __init__(
        self,
        banco: str = "memoria.json",
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

    def _normalizar(self, texto: Optional[str]) -> str:
        if texto is None:
            return ""
        return texto.strip().lower()

    def _carregar_memoria(self) -> Dict[str, Any]:
        with self._lock:
            if os.path.exists(self.banco):
                try:
                    with open(self.banco, "r", encoding="utf-8") as f:
                        data = json.load(f)
                        # garantir chaves normalizadas e metadados
                        normalized = {}
                        for k, v in (data.items() if isinstance(data, dict) else []):
                            nk = self._normalizar(k)
                            normalized[nk] = v
                        self._load_time = datetime.utcnow().isoformat()
                        return normalized
                except Exception as e:
                    print(f"[AIMemoria] Aviso: não foi possível carregar memória ({e}). Iniciando vazia.")
            return {}

    def _salvar_memoria(self) -> None:
        with self._lock:
            if len(self.memoria) > self.max_entries:
                raise RuntimeError("Memória excedeu max_entries")
            tmp_path = self.banco + ".tmp"
            try:
                with open(tmp_path, "w", encoding="utf-8") as f:
                    json.dump(self.memoria, f, indent=2, ensure_ascii=False)
                os.replace(tmp_path, self.banco)
            except Exception as e:
                print(f"[AIMemoria] Erro ao salvar memória: {e}")
                if os.path.exists(tmp_path):
                    try:
                        os.remove(tmp_path)
                    except Exception:
                        pass

    def conversar(self, pergunta: str, ask_if_unknown: bool = True) -> Optional[str]:
        """
        Retorna resposta conhecida. Se desconhecida:
         - se auto_aprender e ask_if_unknown -> pergunta ao usuário e salva (opcionalmente com confirmação)
         - se não, retorna None
        """
        chave = self._normalizar(pergunta)
        if not chave:
            return ""

        # resposta conhecida
        if chave in self.memoria:
            return self.memoria[chave].get("resposta") if isinstance(self.memoria[chave], dict) else self.memoria[chave]

        # desconhecida
        if not ask_if_unknown or not self.auto_aprender:
            return None

        prompt = f"Nunca ouvi isso. O que devo responder para: '{pergunta}'?"
        try:
            resp = self.ask_callback(prompt)
        except KeyboardInterrupt:
            print("\nInterrompido pelo usuário.")
            return None

        if resp is None:
            return None

        # filtrar sensíveis
        if contains_sensitive(pergunta) or contains_sensitive(resp):
            print("[AIMemoria] Aviso: conteúdo parece conter dados sensíveis. Não vou salvar automaticamente.")
            # ainda assim podemos retornar resposta sem salvar
            return resp

        # confirmar salvamento se configurado
        if self.confirmar_salvamento:
            conf = self.ask_callback(f"Deseja salvar essa resposta para '{pergunta}'? (s/N)").strip().lower()
            if not conf.startswith("s"):
                return resp

        # salvar com metadados
        item = {
            "resposta": resp,
            "aprendido_em": datetime.utcnow().isoformat(),
            "fonte": "usuario_interativo",
        }

        self.memoria[chave] = item
        try:
            self._salvar_memoria()
        except Exception as e:
            print(f"[AIMemoria] Erro ao persistir: {e}")
        return resp

    def ensinar(self, pergunta: str, resposta: str, salvar: bool = True) -> None:
        """Ensina explicitamente (força o aprendizado)."""
        chave = self._normalizar(pergunta)
        if not chave:
            raise ValueError("Pergunta inválida")

        if contains_sensitive(pergunta) or contains_sensitive(resposta):
            raise ValueError("Conteúdo parece sensível — não é permitido salvar.")

        item = {
            "resposta": resposta,
            "aprendido_em": datetime.utcnow().isoformat(),
            "fonte": "ensinar_api",
        }
        self.memoria[chave] = item
        if salvar:
            self._salvar_memoria()
        print(f"[AIMemoria] Aprendi a responder '{pergunta}' com '{resposta}'")

    def apagar(self, pergunta: str) -> bool:
        """Apaga uma entrada; retorna True se removida."""
        chave = self._normalizar(pergunta)
        if chave in self.memoria:
            del self.memoria[chave]
            self._salvar_memoria()
            return True
        return False

    def procurar(self, termo: str) -> Dict[str, Any]:
        """Procura entradas que contenham o termo (após normalização)."""
        t = self._normalizar(termo)
        return {k: v for k, v in self.memoria.items() if t in k}

    def exportar(self, caminho: str) -> None:
        """Exporta a memória completa para outro arquivo JSON."""
        with self._lock:
            with open(caminho, "w", encoding="utf-8") as f:
                json.dump(self.memoria, f, indent=2, ensure_ascii=False)

    def importar(self, caminho: str, override: bool = False) -> None:
        """Importa um arquivo JSON. Se override=True substitui, senão mescla sem sobrescrever chaves existentes."""
        with self._lock:
            with open(caminho, "r", encoding="utf-8") as f:
                data = json.load(f)
            norm = {self._normalizar(k): v for k, v in data.items()}
            if override:
                self.memoria = norm
            else:
                for k, v in norm.items():
                    if k not in self.memoria:
                        self.memoria[k] = v
            self._salvar_memoria()

eu ainda quero trazer vida a isso como implementar a um sistema operacional ou ate fazer uma conexao com um navegador para que ela tenha aceso a mais dados e conhecimento.
sera que e posivel?
# integração_simplificada.py
from playwright.sync_api import sync_playwright
from bs4 import BeautifulSoup
from ai_memoria import AIMemoria  # seu arquivo criado
import time
import urllib.parse

def extrair_texto_html(html):
    soup = BeautifulSoup(html, "html.parser")
    # heurística: pegar <article>,  <p> relevantes
    article = soup.find("article")
    if article:
        return article.get_text(separator="\n").strip()
    paragraphs = soup.find_all("p")
    text = "\n".join(p.get_text().strip() for p in paragraphs[:6])
    return text.strip()

def buscar_e_aprender(query, ai: AIMemoria, max_sites=3):
    # usar um search engine via URL
    q = urllib.parse.quote_plus(query)
    search_url = f"https://www.bing.com/search?q={q}"
    with sync_playwright() as pw:
        browser = pw.chromium.launch(headless=True)
        page = browser.new_page(user_agent="MyAI/1.0 (+https://example.com)")
        page.goto(search_url, timeout=30000)
        time.sleep(1.0)  # pequeno delay
        # select resultado: dependerá do layout; aqui uma heurística
        links = page.eval_on_selector_all("li.b_algo h2 a", "els => els.map(e => e.href)")  # seletor Bing
        count = 0
        for href in links:
            if count >= max_sites:
                break
            try:
                page.goto(href, timeout=30000)
                time.sleep(1.0)
                html = page.content()
                texto = extrair_texto_html(html)
                if texto:
                    # transformar em resposta resumida — aqui pega os primeiros 400 chars
                    resposta = texto[:800].strip()
                    # evitar salvar PII: seu AIMemoria já tem contains_sensitive
                    ai.ensinar(query, resposta)
                    count += 1
            except Exception as e:
                print("erro visitando", href, e)
        browser.close()

Resumo do que o prototipo faz

Recebe uma consulta (texto).
Faz uma busca via Bing (página web de busca — embora eu recomende uso da API para produção).
Usa Dramaturgo (navegador sem cabeça) para abrir os top N resultados.
Extrai o conteúdo principal com BeautifulSoup (heurística simples).
Gera uma resposta/síntese (aqui usamos uma heurística simples — você pode integrar um depósito LLM).
Chama ai_memoria.AIMemoria.ensinar(query, resposta) para salvar a resposta como aprendido.
