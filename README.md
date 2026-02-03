# requirements:
# pip install aiogram==3.* aiosqlite

import re
import asyncio
from dataclasses import dataclass
from typing import List, Tuple, Dict, Optional

from aiogram import Bot, Dispatcher, F
from aiogram.types import Message
from aiogram.filters import Command

TOKEN = 8207394786:AAFTm6OhIYLLJ7hmO0JcKPKunfcWGNN5yvg

# ==========
# MODELOS / PERFIS (MVP manual)
# Você pode ir preenchendo isso aos poucos
# escala 0-5
# ==========
@dataclass
class PlayerProfile:
    serve: int = 3
    return_: int = 3
    consistency: int = 3
    mental: int = 3
    tiebreak: int = 3
    fitness: int = 3

PROFILES: Dict[str, PlayerProfile] = {
    "Novak Djokovic": PlayerProfile(serve=5, return_=5, consistency=5, mental=5, tiebreak=4, fitness=4),
    "Alex de Minaur": PlayerProfile(serve=3, return_=4, consistency=4, mental=4, tiebreak=2, fitness=5),
    # vá adicionando...
}

# ==========
# CONFIG DO BOT
# ==========
GROUP_ENABLED: Dict[int, bool] = {}  # chat_id -> on/off

# ==========
# PARSER DE LISTA
# ==========
MATCH_LINE = re.compile(r"^\s*(.+?)\s+(vs\.?|x|v)\s+(.+?)\s*$", re.IGNORECASE)

def parse_matches(text: str) -> Tuple[str, List[Tuple[str, str]]]:
    """
    Retorna (tour, [(p1, p2), ...])
    Tour inferido pelo bloco: se contém "ATP" / "WTA" na primeira linha, usa isso; senão "TÊNIS".
    """
    lines = [ln.strip() for ln in text.splitlines() if ln.strip()]
    tour = "TÊNIS"
    matches = []
    for ln in lines:
        if ln.upper() in ("ATP", "WTA", "CHALLENGER", "ITF"):
            tour = ln.upper()
            continue
        m = MATCH_LINE.match(ln)
        if m:
            p1, p2 = m.group(1).strip(), m.group(3).strip()
            matches.append((p1, p2))
    return tour, matches

def get_profile(name: str) -> PlayerProfile:
    # fallback neutro (3/5) se não cadastrado
    return PROFILES.get(name, PlayerProfile())

# ==========
# ENGINE DE SCORING (heurística simples, ajustável)
# ==========
def confidence_score(fav: PlayerProfile, dog: PlayerProfile) -> int:
    """
    Confiança no favorito (0-100) baseado em diferença de atributos.
    """
    # pesos (ajustáveis)
    w = {
        "serve": 1.0,
        "return_": 1.2,
        "consistency": 1.4,
        "mental": 1.0,
        "fitness": 0.9,
        "tiebreak": 0.5,
    }
    def s(p: PlayerProfile) -> float:
        return (
            p.serve*w["serve"]
            + p.return_*w["return_"]
            + p.consistency*w["consistency"]
            + p.mental*w["mental"]
            + p.fitness*w["fitness"]
            + p.tiebreak*w["tiebreak"]
        )

    diff = s(fav) - s(dog)          # tipicamente entre -?? e +??
    # normalização simples
    score = 50 + diff * 6.5         # ajusta sensibilidade
    return max(0, min(100, int(round(score))))

def variance_score(p1: PlayerProfile, p2: PlayerProfile) -> int:
    """
    Risco/variância (0-100): quanto mais, mais perigoso.
    Heurística: inconsistência + tie-break + equilíbrio geral -> mais variância.
    """
    # equilíbrio (diferença pequena = mais variância)
    balance = 5 - abs((p1.serve+p1.return_+p1.consistency) - (p2.serve+p2.return_+p2.consistency)) / 3
    balance = max(0, min(5, balance))

    # tb alto em ambos => jogo mais “apertado” -> mais variância, e candidato a overs
    tb = (p1.tiebreak + p2.tiebreak) / 2

    # inconsistência alta aumenta variância
    incons = ( (5 - p1.consistency) + (5 - p2.consistency) ) / 2

    raw = (balance*12) + (tb*8) + (incons*12)
    return max(0, min(100, int(round(raw))))

def suggest_tags(p1: PlayerProfile, p2: PlayerProfile) -> List[str]:
    tags = []
    if (p1.tiebreak + p2.tiebreak) >= 8:
        tags.append("tie-break friendly")
        tags.append("over games candidato")
    if abs(p1.serve - p2.serve) <= 1 and abs(p1.return_ - p2.return_) <= 1:
        tags.append("equilíbrio técnico")
        tags.append("3-sets provável")
    if (5 - p1.consistency) + (5 - p2.consistency) >= 5:
        tags.append("inconsistência alta")
    return tags

def label_from_scores(conf: int, var: int) -> str:
    # simples e eficaz
    if conf >= 70 and var <= 55:
        return "✅ Confiável"
    if conf >= 60 and var <= 70:
        return "⚠️ Médio"
    return "❌ Evitar"

# ==========
# TELEGRAM BOT
# ==========
bot = Bot(token=TOKEN)
dp = Dispatcher()

@dp.message(Command("start"))
async def start(message: Message):
    await message.reply(
        "✅ Bot de análise (sem odds)\n\n"
        "Comandos:\n"
        "/on  — ativar análises no grupo\n"
        "/off — pausar\n"
        "/analisar — cole a lista de jogos abaixo do comando\n\n"
        "Formato: 'Jogador A vs Jogador B' (uma por linha)\n"
        "Você pode colocar 'ATP' ou 'WTA' numa linha separada."
    )

@dp.message(Command("on"))
async def on_cmd(message: Message):
    GROUP_ENABLED[message.chat.id] = True
    await message.reply("✅ Ativado: vou responder análises quando você usar /analisar.")

@dp.message(Command("off"))
async def off_cmd(message: Message):
    GROUP_ENABLED[message.chat.id] = False
    await message.reply("⏸️ Pausado: não vou processar análises até você usar /on.")

@dp.message(Command("regras"))
async def rules(message: Message):
    await message.reply(
        "Regras atuais (MVP):\n"
        "- Confiança = diferença de perfis (serve/return/consistência/mental/fitness)\n"
        "- Variância = equilíbrio técnico + tie-break + inconsistência\n"
        "- Saída: ✅ / ⚠️ / ❌\n\n"
        "Obs: Perfis são manuais (você vai cadastrando jogadores aos poucos)."
    )

@dp.message(Command("analisar"))
async def analyze(message: Message):
    if GROUP_ENABLED.get(message.chat.id, True) is False:
        return

    text = message.text or ""
    # remove o comando
    payload = text.split("\n", 1)
    if len(payload) == 1:
        await message.reply("Cole a lista de jogos abaixo do /analisar.\nEx:\n/analisar\nATP\nA vs B\nC vs D")
        return

    tour, matches = parse_matches(payload[1])
    if not matches:
        await message.reply("Não encontrei jogos no formato 'A vs B'.")
        return

    results = []
    for a, b in matches:
        pa, pb = get_profile(a), get_profile(b)

        # favorito = o que tem maior “força base” (simplificado)
        # você pode trocar por ranking manual depois
        base_a = pa.serve + pa.return_ + pa.consistency + pa.mental + pa.fitness
        base_b = pb.serve + pb.return_ + pb.consistency + pb.mental + pb.fitness

        fav_name, dog_name = (a, b) if base_a >= base_b else (b, a)
        fav, dog = (pa, pb) if base_a >= base_b else (pb, pa)

        conf = confidence_score(fav, dog)
        var = variance_score(pa, pb)
        tags = suggest_tags(pa, pb)
        label = label_from_scores(conf, var)

        results.append((conf, var, fav_name, dog_name, label, tags))

    # ranking por confiança (desc) e depois menor variância
    results.sort(key=lambda x: (-x[0], x[1]))

    top = results[:10]
    lines = [f"🎾 {tour} — Ranking de Confiabilidade (sem odds)\n"]
    for i, (conf, var, fav, dog, label, tags) in enumerate(top, start=1):
        tag_txt = " | ".join(tags[:3]) if tags else "—"
        lines.append(
            f"{i}) {fav} vs {dog}\n"
            f"   Confiança: {conf} | Risco: {var} → {label}\n"
            f"   Tags: {tag_txt}\n"
        )

    await message.reply("\n".join(lines))

async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
