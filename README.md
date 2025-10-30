# teste-de-AI
essa foi uma tentativa de criação de uma ai  onde ela vai  aprendendo com a pessoa que vai utilizando e se aprimorando com o tempo sozinha.



import json
import os

class LekDoCérebro:
    def init(self, base_de_dados='lek_memoria.json'):
        self.banco = base_de_dados
        self.memoria = self._carregar_memoria()

    def _carregar_memoria(self):
        if os.path.exists(self.banco):
            with open(self.banco, 'r', encoding='utf-8') as file:
                return json.load(file)
        return {}

    def _salvar_memoria(self):
        with open(self.banco, 'w', encoding='utf-8') as file:
            json.dump(self.memoria, file, indent=4)

    def conversar(self, pergunta):
        pergunta = pergunta.lower()
        if pergunta in self.memoria:
            return self.memoria[pergunta]
        else:
            resposta = input(f"Nunca ouvi isso. O que devo responder pra: '{pergunta}'?\n> ")
            self.memoria[pergunta] = resposta
            self._salvar_memoria()
            return resposta

    def ensinar(self, pergunta, resposta):
        self.memoria[pergunta.lower()] = resposta
        self._salvar_memoria()
        print(f"Aprendi a responder '{pergunta}' com '{resposta}'")

lek = LekDoCérebro()

# Loop doentio de conversa
print("🤖 LEK DO CÉREBRO ATIVADO — FALA ALGUMA COISA:")
while True:
    user_input = input("Você: ")
    if user_input.lower() in ['sair', 'tchau', 'desliga']:
        print("FINALIZANDO... VAZA ENTÃO 👋")
        break
    resposta = lek.conversar(user_input)
    print("Lek:", resposta)


    pip install fastapi uvicorn faiss-cpu sentence-transformers 
    from fastapi import FastAPI, Request
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np
import json
import os

app = FastAPI()
modelo = SentenceTransformer('paraphrase-MiniLM-L6-v2')  # Pequeno, rápido e violento
respostas = []
perguntas = []

# Banco de vetores
dim = 384  # depende do modelo
index = faiss.IndexFlatL2(dim)

# Carregar se já existir
if os.path.exists('lek_memoria.json'):
    with open('lek_memoria.json', 'r', encoding='utf-8') as f:
        dados = json.load(f)
        perguntas = list(dados.keys())
        respostas = list(dados.values())
        vetores = modelo.encode(perguntas)
        index.add(np.array(vetores, dtype=np.float32))

# Ensinador
@app.post("/ensinar")
async def ensinar(data: Request):
    corpo = await data.json()
    pergunta = corpo["pergunta"]
    resposta = corpo["resposta"]

    if pergunta not in perguntas:
        perguntas.append(pergunta)
        respostas.append(resposta)
        vetor = modelo.encode([pergunta])
        index.add(np.array(vetor, dtype=np.float32))
        salvar_memoria()

    return {"mensagem": "Aprendido, chefia!"}

# Conversador
@app.get("/perguntar")
async def perguntar(q: str):
    if not perguntas:
        return {"resposta": "Tô com Alzheimer digital... me ensina algo primeiro."}

    vetor = modelo.encode([q])
    D, I = index.search(np.array(vetor, dtype=np.float32), k=1)
    indice = I[0][0]
    distancia = D[0][0]

    if distancia < 1.0:  # quanto menor, mais parecido
        return {"resposta": respostas[indice]}
    else:
        return {"resposta": "Nunca ouvi isso aí, ensina no /ensinar"}

# Salvar memória
def salvar_memoria():
    with open('lek_memoria.json', 'w', encoding='utf-8') as f:
        json.dump(dict(zip(perguntas, respostas)), f, indent=4, ensure_ascii=False)
