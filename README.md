import random
import os
import time

# ─── Cores ANSI ───────────────────────────────────────────────
R  = "\033[91m"   # vermelho
G  = "\033[92m"   # verde
Y  = "\033[93m"   # amarelo
B  = "\033[94m"   # azul
M  = "\033[95m"   # magenta
C  = "\033[96m"   # ciano
W  = "\033[97m"   # branco
DIM = "\033[2m"
BOLD = "\033[1m"
RESET = "\033[0m"

def cls():
    os.system("cls" if os.name == "nt" else "clear")

def pause(msg="Pressione ENTER para continuar..."):
    input(f"\n{DIM}{msg}{RESET}")

def barra(atual, maximo, tamanho=20, cor=G):
    cheio = int((atual / maximo) * tamanho)
    vazio = tamanho - cheio
    return f"{cor}{'█' * cheio}{DIM}{'░' * vazio}{RESET}"

# ─── Classes ──────────────────────────────────────────────────
class Personagem:
    def __init__(self, nome, hp, mp, ataque, defesa, sprite):
        self.nome    = nome
        self.hp      = hp
        self.hp_max  = hp
        self.mp      = mp
        self.mp_max  = mp
        self.ataque  = ataque
        self.defesa  = defesa
        self.sprite  = sprite
        self.status  = []   # lista de dicts {tipo, duracao}

    def vivo(self):
        return self.hp > 0

    def aplicar_dano(self, dmg):
        dmg_real = max(1, dmg - self.defesa // 2)
        self.hp  = max(0, self.hp - dmg_real)
        return dmg_real

    def curar(self, valor):
        antes = self.hp
        self.hp = min(self.hp_max, self.hp + valor)
        return self.hp - antes

    def tem_status(self, tipo):
        return any(s["tipo"] == tipo for s in self.status)

    def adicionar_status(self, tipo, duracao):
        if not self.tem_status(tipo):
            self.status.append({"tipo": tipo, "duracao": duracao})

    def tick_status(self):
        msgs = []
        novos = []
        for s in self.status:
            if s["tipo"] == "veneno":
                dmg = max(1, self.hp_max // 10)
                self.hp = max(0, self.hp - dmg)
                msgs.append(f"{R}☠ {self.nome} sofre {dmg} de veneno!{RESET}")
            if s["tipo"] == "regen":
                hp = self.hp_max // 8
                self.hp = min(self.hp_max, self.hp + hp)
                msgs.append(f"{G}✨ {self.nome} regenera {hp} de vida.{RESET}")
            s["duracao"] -= 1
            if s["duracao"] > 0:
                novos.append(s)
        self.status = novos
        return msgs

    def tags_status(self):
        nomes = {"veneno": f"{M}[Veneno]{RESET}", "stun": f"{Y}[Atordoado]{RESET}", "regen": f"{G}[Regen]{RESET}"}
        return " ".join(nomes.get(s["tipo"], "") for s in self.status)

    def mostrar(self, lado="esq"):
        tags = self.tags_status()
        cor_nome = C if lado == "esq" else R
        print(f"  {cor_nome}{BOLD}{self.sprite} {self.nome}{RESET} {tags}")
        print(f"  ❤  {barra(self.hp, self.hp_max, cor=R)} {self.hp}/{self.hp_max}")
        if self.mp_max > 0:
            print(f"  💎 {barra(self.mp, self.mp_max, cor=B)} {self.mp}/{self.mp_max}")


class Heroi(Personagem):
    def __init__(self):
        super().__init__("Aldric", 80, 40, 15, 8, "🧙")
        self.nivel   = 1
        self.xp      = 0
        self.xp_prox = 100
        self.pocoes  = 2
        self.ouro    = 0
        self.defendendo = False

    def mostrar(self, lado="esq"):
        super().mostrar(lado)
        print(f"  ✨ XP: {self.xp}/{self.xp_prox}  |  🪙 Ouro: {self.ouro}  |  🧪 Poções: {self.pocoes}")

    def ganhar_xp(self, valor):
        self.xp += valor
        msgs = []
        while self.xp >= self.xp_prox:
            self.xp      -= self.xp_prox
            self.nivel   += 1
            self.xp_prox  = int(self.xp_prox * 1.6)
            self.hp_max  += 20
            self.hp       = self.hp_max
            self.mp_max  += 10
            self.mp       = self.mp_max
            self.ataque  += 5
            self.defesa  += 2
            self.pocoes  += 1
            msgs.append(f"{Y}{BOLD}★ NÍVEL {self.nivel}! Você ficou mais forte! (+poção){RESET}")
        return msgs


class Inimigo(Personagem):
    def __init__(self, nome, hp, mp, ataque, defesa, sprite, xp, ouro, nivel, habilidades):
        super().__init__(nome, hp, mp, ataque, defesa, sprite)
        self.xp_recompensa   = xp
        self.ouro_recompensa = ouro
        self.nivel           = nivel
        self.habilidades     = habilidades  # lista de funções

    def agir(self, heroi):
        if self.tem_status("stun"):
            return [f"{Y}💫 {self.nome} está atordoado e perde o turno!{RESET}"]
        habilidade = random.choice(self.habilidades)
        return habilidade(self, heroi)


# ─── Habilidades dos inimigos ─────────────────────────────────
def ataque_basico(inimigo, heroi):
    dmg = random.randint(int(inimigo.ataque * 0.85), int(inimigo.ataque * 1.15))
    real = heroi.aplicar_dano(dmg) if not heroi.defendendo else max(1, dmg // 3)
    if heroi.defendendo:
        heroi.hp = max(0, heroi.hp - real)
        return [f"{R}💥 {inimigo.nome} ataca por {dmg}! Bloqueado → {real} de dano.{RESET}"]
    heroi.hp = max(0, heroi.hp - real + heroi.defesa // 2)
    real2 = heroi.aplicar_dano(dmg)
    return [f"{R}💥 {inimigo.nome} ataca {heroi.nome} por {real2} de dano!{RESET}"]

def mordida_venenosa(inimigo, heroi):
    msgs = ataque_basico(inimigo, heroi)
    if random.random() < 0.5 and not heroi.tem_status("veneno"):
        heroi.adicionar_status("veneno", 3)
        msgs.append(f"{M}🐍 {heroi.nome} foi envenenado!{RESET}")
    return msgs

def rugido(inimigo, heroi):
    heroi.ataque = max(5, heroi.ataque - 3)
    return [f"{Y}🦁 {inimigo.nome} ruge! Seu ataque caiu por 3 por intimidação!{RESET}"]

def golpe_pesado(inimigo, heroi):
    dmg = int(inimigo.ataque * 1.6)
    real = heroi.aplicar_dano(dmg)
    msgs = [f"{R}🔨 {inimigo.nome} usa Golpe Pesado! -{real} de vida!{RESET}"]
    if random.random() < 0.3:
        heroi.adicionar_status("stun", 1)
        msgs.append(f"{Y}💫 Você foi atordoado!{RESET}")
    return msgs

def baforada_de_fogo(inimigo, heroi):
    dmg = int(inimigo.ataque * 1.4)
    real = heroi.aplicar_dano(dmg)
    return [f"{R}🔥 {inimigo.nome} cuspiu fogo! -{real} de dano mágico!{RESET}"]


# ─── Catálogo de inimigos ─────────────────────────────────────
INIMIGOS = [
    lambda: Inimigo("Goblin",        60,  0, 12,  3, "👺",  80, 15, 1, [ataque_basico, mordida_venenosa]),
    lambda: Inimigo("Orc Guerreiro",100,  0, 18,  6, "🧌", 130, 25, 2, [ataque_basico, golpe_pesado, rugido]),
    lambda: Inimigo("Troll",        160,  0, 26, 10, "👹", 200, 40, 3, [ataque_basico, golpe_pesado, golpe_pesado]),
    lambda: Inimigo("Dragão Negro", 280, 60, 38, 14, "🐉", 350, 80, 5, [ataque_basico, baforada_de_fogo, golpe_pesado, mordida_venenosa]),
]


# ─── Tela de batalha ──────────────────────────────────────────
def desenhar_tela(heroi, inimigo, turno, mensagens):
    cls()
    print(f"\n{Y}{BOLD}  ⚔  CHRONICLES OF FATE  ⚔{RESET}")
    print(f"  {DIM}{'─'*44}{RESET}\n")

    print(f"{C}  [ HERÓI ]{RESET}")
    heroi.mostrar("esq")
    print()
    print(f"{R}  [ INIMIGO — Lv.{inimigo.nivel} ]{RESET}")
    inimigo.mostrar("dir")
    print(f"\n  {DIM}{'─'*44}{RESET}")

    for msg in mensagens[-5:]:
        print(f"  {msg}")

    print(f"\n  {DIM}{'─'*44}{RESET}")
    if turno == "heroi":
        print(f"\n{W}  {BOLD}[1]{RESET} ⚔  Atacar")
        print(f"{W}  {BOLD}[2]{RESET} 🔥  Bola de Fogo   {DIM}(20 mana){RESET}")
        print(f"{W}  {BOLD}[3]{RESET} 🧊  Toque Gelado   {DIM}(15 mana — atordoa){RESET}")
        print(f"{W}  {BOLD}[4]{RESET} 💊  Usar Poção     {DIM}({heroi.pocoes} restantes){RESET}")
        print(f"{W}  {BOLD}[5]{RESET} 🛡  Defender       {DIM}(reduz dano, +mana){RESET}")


def escolher_acao(heroi):
    while True:
        escolha = input(f"\n  {C}> Sua ação: {RESET}").strip()
        if escolha == "1":
            return "atacar"
        if escolha == "2":
            if heroi.mp < 20:
                print(f"  {R}Mana insuficiente!{RESET}")
            else:
                return "magia"
        if escolha == "3":
            if heroi.mp < 15:
                print(f"  {R}Mana insuficiente!{RESET}")
            else:
                return "gelo"
        if escolha == "4":
            if heroi.pocoes <= 0:
                print(f"  {R}Sem poções!{RESET}")
            else:
                return "pocao"
        if escolha == "5":
            return "defender"


# ─── Turno do herói ───────────────────────────────────────────
def turno_heroi(heroi, inimigo, msgs):
    acao = escolher_acao(heroi)
    heroi.defendendo = False

    if acao == "atacar":
        crit  = random.random() < 0.2
        mult  = 1.5 if crit else 1.0
        dmg   = int(random.randint(int(heroi.ataque * 0.85), int(heroi.ataque * 1.15)) * mult)
        real  = inimigo.aplicar_dano(dmg)
        prefixo = f"{Y}★ CRÍTICO!{RESET} " if crit else ""
        msgs.append(f"{C}⚔ {prefixo}{heroi.nome} ataca por {real} de dano!{RESET}")
        if random.random() < 0.2 and not inimigo.tem_status("veneno"):
            inimigo.adicionar_status("veneno", 3)
            msgs.append(f"{M}🧪 {inimigo.nome} foi envenenado!{RESET}")

    elif acao == "magia":
        heroi.mp -= 20
        dmg  = int(heroi.ataque * 1.5) + random.randint(5, 15)
        real = inimigo.aplicar_dano(dmg)
        msgs.append(f"{R}🔥 Bola de Fogo! {inimigo.nome} recebe {real} de dano mágico!{RESET}")

    elif acao == "gelo":
        heroi.mp -= 15
        dmg  = int(heroi.ataque * 0.9)
        real = inimigo.aplicar_dano(dmg)
        msgs.append(f"{B}🧊 Toque Gelado! {inimigo.nome} recebe {real} de dano!{RESET}")
        if not inimigo.tem_status("stun"):
            inimigo.adicionar_status("stun", 1)
            msgs.append(f"{Y}💫 {inimigo.nome} foi atordoado!{RESET}")

    elif acao == "pocao":
        heroi.pocoes -= 1
        cura = int(heroi.hp_max * 0.45)
        real = heroi.curar(cura)
        heroi.adicionar_status("regen", 2)
        msgs.append(f"{G}💊 Você usou uma poção e recuperou {real} de vida!{RESET}")

    elif acao == "defender":
        heroi.defendendo = True
        mp_rec = heroi.mp_max // 4
        heroi.mp = min(heroi.mp_max, heroi.mp + mp_rec)
        msgs.append(f"{C}🛡 {heroi.nome} se defende! +{mp_rec} mana.{RESET}")


# ─── Loop de batalha ──────────────────────────────────────────
def batalha(heroi, inimigo):
    msgs = [f"{Y}⚔ {inimigo.sprite} {inimigo.nome} (Lv.{inimigo.nivel}) aparece!{RESET}"]
    turno = "heroi"

    while heroi.vivo() and inimigo.vivo():
        desenhar_tela(heroi, inimigo, turno, msgs)

        if turno == "heroi":
            if heroi.tem_status("stun"):
                heroi.status = [s for s in heroi.status if not (s["tipo"]=="stun" and s["duracao"]<=1)]
                msgs.append(f"{Y}💫 Você está atordoado e perde o turno!{RESET}")
                turno = "inimigo"
                time.sleep(1)
                continue
            turno_heroi(heroi, inimigo, msgs)
            msgs += heroi.tick_status()
            turno = "inimigo"
        else:
            msgs += inimigo.tick_status()
            if inimigo.vivo():
                msgs += inimigo.agir(heroi)
            msgs += heroi.tick_status()
            heroi.mp = min(heroi.mp_max, heroi.mp + 5)  # regen passiva de mana
            turno = "heroi"
            if not heroi.vivo():
                break

    desenhar_tela(heroi, inimigo, "", msgs)

    if not heroi.vivo():
        return False

    # Vitória
    print(f"\n  {G}{BOLD}✅ {inimigo.nome} foi derrotado!{RESET}")
    xp_msgs = heroi.ganhar_xp(inimigo.xp_recompensa)
    heroi.ouro += inimigo.ouro_recompensa
    print(f"  {Y}+{inimigo.xp_recompensa} XP  |  🪙 +{inimigo.ouro_recompensa} ouro{RESET}")
    for m in xp_msgs:
        print(f"  {m}")
    return True


# ─── Menu inicial ─────────────────────────────────────────────
def tela_inicial():
    cls()
    print(f"""
{Y}{BOLD}
  ╔══════════════════════════════════════════╗
  ║       ⚔  CHRONICLES OF FATE  ⚔         ║
  ║         RPG de Turnos em Python          ║
  ╚══════════════════════════════════════════╝
{RESET}
{DIM}  Derrote 4 inimigos para salvar o reino.
  Cada vitória concede XP, ouro e poções.

  Habilidades:
   ⚔  Atacar      — dano físico, chance de crítico e veneno
   🔥  Bola de Fogo — dano mágico alto (20 mana)
   🧊  Toque Gelado — dano + atordoa (15 mana)
   💊  Poção        — cura + regeneração por 2 turnos
   🛡  Defender     — reduz dano, regenera mana
{RESET}""")
    input(f"  {C}Pressione ENTER para começar...{RESET}")


# ─── Main ─────────────────────────────────────────────────────
def main():
    while True:
        tela_inicial()
        heroi = Heroi()

        vitoria_total = True
        for fabricar in INIMIGOS:
            inimigo = fabricar()
            if not batalha(heroi, inimigo):
                vitoria_total = False
                break
            if fabricar != INIMIGOS[-1]:
                pause("Próximo inimigo aguarda... ENTER para continuar.")

        cls()
        if vitoria_total:
            print(f"""
{Y}{BOLD}
  ╔══════════════════════════════════════╗
  ║      🏆  VITÓRIA LENDÁRIA!  🏆      ║
  ╚══════════════════════════════════════╝
{RESET}
  {G}Você derrotou o Dragão Negro e salvou o reino!{RESET}

  {C}Nível final : {heroi.nivel}
  Ouro obtido : {heroi.ouro} 🪙
  Vida restante: {heroi.hp}/{heroi.hp_max}{RESET}
""")
        else:
            print(f"""
{R}{BOLD}
  ╔══════════════════════════════════════╗
  ║         💀  GAME OVER  💀           ║
  ╚══════════════════════════════════════╝
{RESET}
  {DIM}Você foi derrotado em batalha...{RESET}

  {C}Nível atingido : {heroi.nivel}
  Ouro obtido    : {heroi.ouro} 🪙{RESET}
""")

        novamente = input(f"  {C}Jogar novamente? (s/n): {RESET}").strip().lower()
        if novamente != "s":
            print(f"\n  {Y}Até a próxima aventura! ⚔{RESET}\n")
            break


if __name__ == "__main__":
    main()