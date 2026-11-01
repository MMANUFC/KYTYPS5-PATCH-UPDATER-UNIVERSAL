# KYTYPS5-PATCH-UPDATER-UNIVERSAL
A simple program, created in 10 minutes with the help of AI, that will save you time. You just provide the old patch (a .json file), the game's `eboot.bin` file, and the name you want for the new patch; the program then updates it to the latest version.
Why share this?
Simply to save you the time and hassle of asking an AI—since offsets change, it’s not enough to just update the version number. I hope you find it useful.

source code: 


#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
ISMANUEEE · Kyty Patch Porter
=============================
Ports any KytyPS5 JSON patch (any game, any version) to another game version.
Porta cualquier parche JSON de KytyPS5 a otra version del juego.

GUI (recommended / recomendado):
    python kyty_patch_porter.py

Console / consola:
    python kyty_patch_porter.py --patch patch.json --new eboot_new.bin \
                                [--old eboot_old.bin] [--out out.json] [--ver 01.006.000] \
                                [--lang es|en|pt|ru]
"""
import argparse
import base64
import copy
import json
import os
import queue
import re
import struct
import sys
import threading

APP = "Kyty Patch Porter"
AUTHOR = "ISMANUEEE"

# Colores estilo Discord
BG = "#1E1F22"
PANEL = "#2B2D31"
PANEL2 = "#313338"
BLURPLE = "#5865F2"
BLURPLE_H = "#4752C4"
TXT = "#DBDEE1"
MUTED = "#949BA4"
GREEN = "#57F287"
YELLOW = "#FEE75C"
RED = "#ED4245"

# =====================================================================
#  Idiomas
# =====================================================================
LANGS = [("es", "Español"), ("en", "English"), ("pt", "Português"), ("ru", "Русский")]
LANG = "es"
CONF_PATH = os.path.join(os.path.expanduser("~"), ".ismanueee_kyty_porter.json")

STR = {
    "es": {
        "sub": "PORTADOR UNIVERSAL DE PARCHES",
        "l_patch": "Parche JSON", "h_patch": "el parche de la versión anterior",
        "l_new": "eboot.bin NUEVO", "h_new": "obligatorio",
        "l_old": "eboot.bin VIEJO", "h_old": "opcional, pero MUCHO más fiable",
        "l_ver": "Versión nueva", "h_ver": "opcional, p. ej. 01.006.000",
        "l_out": "Guardar como", "h_out": "opcional (por defecto: junto al parche)",
        "browse": "Elegir…", "go": "PORTAR PARCHE", "busy": "Trabajando…",
        "footer": "hecho por", "need": "Elige al menos el parche JSON y el eboot.bin nuevo.",
        "dlg_patch": "Parche JSON de Kyty", "dlg_new": "eboot.bin de la versión nueva",
        "dlg_old": "eboot.bin de la versión vieja", "dlg_save": "Guardar parche",
        "ft_all": "Todos", "lang_tip": "Idioma",
        "m_entries": "{n} entradas de memoria ({c} cuevas de código) en {m} parches.",
        "m_oldmis": "  0x{a:08X}: los bytes 'off' no coinciden en el eboot viejo; se ignora el contexto.",
        "m_fail": "  0x{a:08X}: NO SE PUDO LOCALIZAR.",
        "m_outside": "  0x{a:08X}: la posición nueva cae fuera de los segmentos.",
        "m_cand": "      candidato 0x{v:08X} (desplazamiento {dl})",
        "m_cavefree": "  cueva 0x{a:08X}: no hay hueco libre.",
        "m_cave": "  cueva 0x{o:08X} -> 0x{n:08X}  [{how}]",
        "how_same": "misma dirección que en la versión vieja",
        "how_fill": "relleno 0x{f:02X}",
        "how_after": "tras el segmento (arriesgado: depende de que Kyty lo mapee)",
        "m_jump": "  salto en 0x{a:08X}+{i}: recalculado.",
        "m_new": "Eboot nuevo: {name} ({kind}, {size:#x} bytes)",
        "m_old": "Eboot viejo: {name} ({kind}, {size:#x} bytes)",
        "m_noold": "Sin eboot viejo: se usarán firmas y desplazamientos deducidos (menos fiable).",
        "m_saved": "\nGuardado: {p}",
        "m_ported": "Parches portados: {k} de {t}.",
        "m_excl": "  EXCLUIDO (alguna entrada no se pudo localizar): {n}",
        "m_tip": "Pruébalos de uno en uno en Kyty; los marcados [{p}] o [{d}] son los que hay que vigilar.",
        "m_err": "\nERROR: {e}",
        "cli_sub": "portador universal de parches Kyty",
        "noui": "No se pudo abrir la interfaz gráfica ({e}).",
        "c_seguro": "seguro", "c_probable": "probable", "c_dudoso": "dudoso",
    },
    "en": {
        "sub": "UNIVERSAL PATCH PORTER",
        "l_patch": "JSON patch", "h_patch": "the patch from the previous version",
        "l_new": "NEW eboot.bin", "h_new": "required",
        "l_old": "OLD eboot.bin", "h_old": "optional, but MUCH more reliable",
        "l_ver": "New version", "h_ver": "optional, e.g. 01.006.000",
        "l_out": "Save as", "h_out": "optional (default: next to the patch)",
        "browse": "Browse…", "go": "PORT PATCH", "busy": "Working…",
        "footer": "made by", "need": "Pick at least the JSON patch and the new eboot.bin.",
        "dlg_patch": "Kyty JSON patch", "dlg_new": "eboot.bin of the new version",
        "dlg_old": "eboot.bin of the old version", "dlg_save": "Save patch",
        "ft_all": "All files", "lang_tip": "Language",
        "m_entries": "{n} memory entries ({c} code caves) in {m} patches.",
        "m_oldmis": "  0x{a:08X}: the 'off' bytes do not match in the old eboot; context is ignored.",
        "m_fail": "  0x{a:08X}: COULD NOT BE LOCATED.",
        "m_outside": "  0x{a:08X}: the new position falls outside the segments.",
        "m_cand": "      candidate 0x{v:08X} (shift {dl})",
        "m_cavefree": "  cave 0x{a:08X}: no free space.",
        "m_cave": "  cave 0x{o:08X} -> 0x{n:08X}  [{how}]",
        "how_same": "same address as in the old version",
        "how_fill": "padding 0x{f:02X}",
        "how_after": "after the segment (risky: depends on Kyty mapping it)",
        "m_jump": "  jump at 0x{a:08X}+{i}: recalculated.",
        "m_new": "New eboot: {name} ({kind}, {size:#x} bytes)",
        "m_old": "Old eboot: {name} ({kind}, {size:#x} bytes)",
        "m_noold": "No old eboot: signatures and deduced shifts will be used (less reliable).",
        "m_saved": "\nSaved: {p}",
        "m_ported": "Patches ported: {k} of {t}.",
        "m_excl": "  EXCLUDED (an entry could not be located): {n}",
        "m_tip": "Test them one by one in Kyty; the ones marked [{p}] or [{d}] are the ones to watch.",
        "m_err": "\nERROR: {e}",
        "cli_sub": "universal Kyty patch porter",
        "noui": "Could not open the graphical interface ({e}).",
        "c_seguro": "safe", "c_probable": "likely", "c_dudoso": "doubtful",
    },
    "pt": {
        "sub": "PORTADOR UNIVERSAL DE PATCHES",
        "l_patch": "Patch JSON", "h_patch": "o patch da versão anterior",
        "l_new": "eboot.bin NOVO", "h_new": "obrigatório",
        "l_old": "eboot.bin ANTIGO", "h_old": "opcional, mas MUITO mais confiável",
        "l_ver": "Versão nova", "h_ver": "opcional, ex.: 01.006.000",
        "l_out": "Salvar como", "h_out": "opcional (padrão: ao lado do patch)",
        "browse": "Escolher…", "go": "PORTAR PATCH", "busy": "Trabalhando…",
        "footer": "feito por", "need": "Escolha pelo menos o patch JSON e o eboot.bin novo.",
        "dlg_patch": "Patch JSON do Kyty", "dlg_new": "eboot.bin da versão nova",
        "dlg_old": "eboot.bin da versão antiga", "dlg_save": "Salvar patch",
        "ft_all": "Todos", "lang_tip": "Idioma",
        "m_entries": "{n} entradas de memória ({c} cavernas de código) em {m} patches.",
        "m_oldmis": "  0x{a:08X}: os bytes 'off' não coincidem no eboot antigo; o contexto é ignorado.",
        "m_fail": "  0x{a:08X}: NÃO FOI POSSÍVEL LOCALIZAR.",
        "m_outside": "  0x{a:08X}: a nova posição cai fora dos segmentos.",
        "m_cand": "      candidato 0x{v:08X} (deslocamento {dl})",
        "m_cavefree": "  caverna 0x{a:08X}: sem espaço livre.",
        "m_cave": "  caverna 0x{o:08X} -> 0x{n:08X}  [{how}]",
        "how_same": "mesmo endereço da versão antiga",
        "how_fill": "preenchimento 0x{f:02X}",
        "how_after": "após o segmento (arriscado: depende de o Kyty mapear)",
        "m_jump": "  salto em 0x{a:08X}+{i}: recalculado.",
        "m_new": "Eboot novo: {name} ({kind}, {size:#x} bytes)",
        "m_old": "Eboot antigo: {name} ({kind}, {size:#x} bytes)",
        "m_noold": "Sem eboot antigo: serão usadas assinaturas e deslocamentos deduzidos (menos confiável).",
        "m_saved": "\nSalvo: {p}",
        "m_ported": "Patches portados: {k} de {t}.",
        "m_excl": "  EXCLUÍDO (alguma entrada não pôde ser localizada): {n}",
        "m_tip": "Teste um por um no Kyty; os marcados [{p}] ou [{d}] são os que precisam de atenção.",
        "m_err": "\nERRO: {e}",
        "cli_sub": "portador universal de patches Kyty",
        "noui": "Não foi possível abrir a interface gráfica ({e}).",
        "c_seguro": "seguro", "c_probable": "provável", "c_dudoso": "duvidoso",
    },
    "ru": {
        "sub": "УНИВЕРСАЛЬНЫЙ ПЕРЕНОС ПАТЧЕЙ",
        "l_patch": "Патч JSON", "h_patch": "патч предыдущей версии",
        "l_new": "НОВЫЙ eboot.bin", "h_new": "обязательно",
        "l_old": "СТАРЫЙ eboot.bin", "h_old": "необязательно, но намного надёжнее",
        "l_ver": "Новая версия", "h_ver": "необязательно, напр. 01.006.000",
        "l_out": "Сохранить как", "h_out": "необязательно (по умолчанию: рядом с патчем)",
        "browse": "Выбрать…", "go": "ПЕРЕНЕСТИ ПАТЧ", "busy": "Работаю…",
        "footer": "сделано", "need": "Выберите как минимум патч JSON и новый eboot.bin.",
        "dlg_patch": "Патч JSON для Kyty", "dlg_new": "eboot.bin новой версии",
        "dlg_old": "eboot.bin старой версии", "dlg_save": "Сохранить патч",
        "ft_all": "Все файлы", "lang_tip": "Язык",
        "m_entries": "{n} записей памяти ({c} кодовых пещер) в {m} патчах.",
        "m_oldmis": "  0x{a:08X}: байты 'off' не совпадают в старом eboot; контекст игнорируется.",
        "m_fail": "  0x{a:08X}: НЕ УДАЛОСЬ НАЙТИ.",
        "m_outside": "  0x{a:08X}: новая позиция находится вне сегментов.",
        "m_cand": "      кандидат 0x{v:08X} (смещение {dl})",
        "m_cavefree": "  пещера 0x{a:08X}: нет свободного места.",
        "m_cave": "  пещера 0x{o:08X} -> 0x{n:08X}  [{how}]",
        "how_same": "тот же адрес, что и в старой версии",
        "how_fill": "заполнитель 0x{f:02X}",
        "how_after": "после сегмента (рискованно: зависит от того, отображает ли его Kyty)",
        "m_jump": "  переход в 0x{a:08X}+{i}: пересчитан.",
        "m_new": "Новый eboot: {name} ({kind}, {size:#x} байт)",
        "m_old": "Старый eboot: {name} ({kind}, {size:#x} байт)",
        "m_noold": "Нет старого eboot: будут использованы сигнатуры и выведенные смещения (менее надёжно).",
        "m_saved": "\nСохранено: {p}",
        "m_ported": "Перенесено патчей: {k} из {t}.",
        "m_excl": "  ИСКЛЮЧЁН (не удалось найти одну из записей): {n}",
        "m_tip": "Проверяйте по одному в Kyty; особое внимание на отмеченные [{p}] и [{d}].",
        "m_err": "\nОШИБКА: {e}",
        "cli_sub": "универсальный перенос патчей Kyty",
        "noui": "Не удалось открыть графический интерфейс ({e}).",
        "c_seguro": "надёжно", "c_probable": "вероятно", "c_dudoso": "сомнительно",
    },
}


def t(key, **kw):
    s = STR.get(LANG, STR["es"]).get(key) or STR["es"][key]
    return s.format(**kw) if kw else s


def load_lang():
    global LANG
    try:
        with open(CONF_PATH, "r", encoding="utf-8") as fh:
            v = json.load(fh).get("lang")
        if v in STR:
            LANG = v
    except (OSError, ValueError):
        pass


def save_lang():
    try:
        with open(CONF_PATH, "w", encoding="utf-8") as fh:
            json.dump({"lang": LANG}, fh)
    except OSError:
        pass


# =====================================================================
#  Imagen del ejecutable (SELF / ELF / crudo) -> traduccion fichero <-> direccion virtual
# =====================================================================
SELF_MAGICS = (bytes.fromhex("5414F5EE"), bytes.fromhex("4F153D1D"))


class Image:
    def __init__(self, data, name=""):
        self.d = data
        self.name = name
        self.kind = "raw"
        self.segs = []  # (offset_fichero, vaddr, tamano, ejecutable)
        try:
            if data[:4] == b"\x7fELF":
                self.kind = "ELF"
                self._parse_elf()
            elif data[:4] in SELF_MAGICS:
                self.kind = "SELF"
                self._parse_self()
        except (struct.error, IndexError):
            self.segs = []
        if not self.segs:
            self.kind = "raw"
            self.segs = [(0, 0, len(data), True)]

    @staticmethod
    def _phdrs(d, base):
        (phoff,) = struct.unpack_from("<Q", d, base + 0x20)
        phentsize, phnum = struct.unpack_from("<HH", d, base + 0x36)
        return [struct.unpack_from("<IIQQQQQQ", d, base + phoff + i * phentsize) for i in range(phnum)]

    def _parse_elf(self):
        for p in self._phdrs(self.d, 0):
            if p[0] == 1:  # PT_LOAD
                self.segs.append((p[2], p[3], p[5], bool(p[1] & 1)))

    def _parse_self(self):
        d = self.d
        (nent,) = struct.unpack_from("<H", d, 0x18)
        entries = [struct.unpack_from("<QQQQ", d, 0x20 + i * 32) for i in range(nent)]
        elf0 = 0x20 + nent * 32
        if d[elf0:elf0 + 4] != b"\x7fELF":
            return
        ph = self._phdrs(d, elf0)
        for fl, off, fsz, msz in entries:
            if fl & 0x800:
                sid = (fl >> 20) & 0xFFF
                if sid < len(ph):
                    self.segs.append((off, ph[sid][3], fsz, bool(ph[sid][1] & 1)))

    def to_vaddr(self, f):
        for fo, va, sz, ex in self.segs:
            if fo <= f < fo + sz:
                return va + (f - fo)
        return None

    def to_file(self, v):
        for fo, va, sz, ex in self.segs:
            if va <= v < va + sz:
                return fo + (v - va)
        return None


# =====================================================================
#  Busqueda
# =====================================================================
CTX = 40
ANCHOR = 8


def find_all(data, pat, lo, hi, limit):
    out, s = [], lo
    while len(out) <= limit:
        i = data.find(pat, s, hi)
        if i < 0:
            break
        out.append(i)
        s = i + 1
    return out


def score_at(ctx, b, nd, pos):
    s = pos - b
    if s < 0 or s + len(ctx) > len(nd):
        return 0.0
    w = nd[s:s + len(ctx)]
    return sum(1 for x, y in zip(ctx, w) if x == y) / len(ctx)


def search_ctx(nd, ctx, b, thr, guess_f):
    """Busca 'ctx' (con el parche en la posicion b) tolerando bytes que cambian.
    Devuelve (pos_parche, score, segundo_score) o None."""
    anchors = []
    for j in range(0, len(ctx) - ANCHOR + 1, 2):
        a = ctx[j:j + ANCHOR]
        if len(set(a)) > 2:
            anchors.append((j, a))
    if not anchors:
        return None
    windows = []
    if guess_f is not None:
        windows.append((max(0, guess_f - 0x20000), min(len(nd), guess_f + 0x20000)))
        windows.append((max(0, guess_f - 0x400000), min(len(nd), guess_f + 0x400000)))
    windows.append((0, len(nd)))
    for lo, hi in windows:
        votes = {}
        for j, a in anchors:
            hits = find_all(nd, a, lo, hi, 300)
            if len(hits) > 300:
                continue
            for h in hits:
                p = h - j + b
                votes[p] = votes.get(p, 0) + 1
        if not votes:
            continue
        top = sorted(votes.items(), key=lambda kv: -kv[1])[:20]
        sc = sorted(((score_at(ctx, b, nd, p), p) for p, _ in top), reverse=True)
        best = sc[0]
        second = sc[1][0] if len(sc) > 1 else 0.0
        if best[0] >= thr and (len(sc) == 1 or best[0] - second >= 0.10):
            return best[1], best[0], second
    return None


def is_filler(b):
    return len(b) >= 16 and len(set(b)) == 1 and b[0] in (0x00, 0xCC, 0x90)


def locate_noold(new, e, nd, solved, log):
    """Sin eboot viejo (o si el contexto fallo): firma larga o desplazamientos deducidos."""
    d, off, old_a = new.d, e["off"], e["old"]
    L = len(off)
    gf = new.to_file(old_a + nd) if nd is not None else None
    if L >= 12 and len(set(off)) > 2:
        hits = find_all(d, off, 0, len(d), 50)
        if len(hits) == 1:
            return hits[0], "seguro"
        if len(hits) > 1 and gf is not None:
            return min(hits, key=lambda h: abs(h - gf)), "probable"
        if not hits:
            r = search_ctx(d, off, 0, 0.80, gf)
            if r:
                return r[0], "probable"
    deltas, seen = [], set()
    if nd is not None:
        deltas.append(nd)
        seen.add(nd)
    for o, n in sorted(solved, key=lambda s: abs(s[0] - old_a)):
        if n - o not in seen:
            seen.add(n - o)
            deltas.append(n - o)
    for dl in deltas:
        f = new.to_file(old_a + dl)
        if f is not None and d[f:f + L] == off:
            return f, ("dudoso" if L <= 2 else "probable")
    if gf is None or L < 4:
        return None  # patrones de 1-3 bytes: demasiado ambiguos para buscarlos a ciegas
    lo, hi = max(0, gf - 0x40000), min(len(d), gf + 0x40000)
    pat = re.compile(re.escape(off))
    cands = []
    for m in pat.finditer(d, lo, hi):
        v = new.to_vaddr(m.start())
        if v is None:
            continue
        dl = v - old_a
        if dl % 16 == 0 and abs(dl - nd) <= 0x30000:
            cands.append((m.start(), dl))
    if len(cands) == 1:
        return cands[0][0], ("dudoso" if L <= 2 else "probable")
    for f, dl in cands[:5]:
        log(t("m_cand", v=new.to_vaddr(f), dl=f"{dl:+#x}"), "warn")
    return None


def find_free(img, size, used):
    """Hueco libre (relleno CC/90/00) de 'size' bytes en segmentos ejecutables.
    Devuelve (vaddr, texto_descriptivo) o None."""
    need = size + 16
    for fill in (0xCC, 0x90, 0x00):
        pat = re.compile(re.escape(bytes([fill])) + b"{%d,}" % need)
        for fo, sz in ((s[0], s[2]) for s in img.segs if s[3]):
            runs = sorted(((m.start(), m.end()) for m in pat.finditer(img.d, fo, fo + sz)),
                          key=lambda r: -(r[1] - r[0]))
            for rs, re_ in runs:
                s = (rs + 0xF) & ~0xF
                while s + size <= re_:
                    v = img.to_vaddr(s)
                    clash = [u for u in used if v < u[1] and v + size > u[0]]
                    if not clash:
                        return v, t("how_fill", f=fill)
                    s = (max(u[1] for u in clash) - v + s + 0xF) & ~0xF
    for fo, va, sz, ex in img.segs:
        if ex:
            end = va + sz
            page = (end + 0x3FFF) // 0x4000 * 0x4000
            v = (end + 0xF) & ~0xF
            if page - v >= size and not [u for u in used if v < u[1] and v + size > u[0]]:
                return v, t("how_after")
    return None


# =====================================================================
#  Motor
# =====================================================================
def key_of(m):
    return (str(m["offset"]).lower(), m["off"].replace(" ", "").upper(), m["on"].replace(" ", "").upper())


def port_patch(patch, new, old=None, log=None, progress=None):
    log = log or (lambda m, lv="info": print(m))
    ents = {}
    for mod in patch.get("mods", []):
        for m in mod.get("memory", []):
            k = key_of(m)
            if k not in ents:
                ents[k] = {"key": k, "old": int(str(m["offset"]), 16),
                           "off": bytes.fromhex(k[1]), "on": bytes.fromhex(k[2]), "new": None}
    caves = [e for e in ents.values() if is_filler(e["off"]) and not is_filler(e["on"])]
    normals = sorted((e for e in ents.values() if e not in caves), key=lambda e: e["old"])
    log(t("m_entries", n=len(ents), c=len(caves), m=len(patch.get("mods", []))))
    done, total = 0, len(ents)
    solved = []

    def neighbor(addr):
        if not solved:
            return None
        o, n = min(solved, key=lambda s: abs(s[0] - addr))
        return n - o

    # ---- 1) entradas normales
    for e in normals:
        done += 1
        if progress:
            progress(done, total)
        L, nd = len(e["off"]), neighbor(e["old"])
        res = None
        if old is not None:
            fo = old.to_file(e["old"])
            if fo is not None and fo + L <= len(old.d):
                if old.d[fo:fo + L] != e["off"]:
                    log(t("m_oldmis", a=e["old"]), "warn")
                else:
                    gf = new.to_file(e["old"] + nd) if nd is not None else None
                    b = min(CTX, fo)
                    a = min(CTX, len(old.d) - fo - L)
                    ctx = old.d[fo - b: fo + L + a]
                    r = search_ctx(new.d, ctx, b, 0.75, gf)
                    if r:
                        res = (r[0], "seguro" if r[1] >= 0.92 else "probable")
        if res is None:
            res = locate_noold(new, e, nd, solved, log)
        if res is None:
            e["status"] = "fallo"
            log(t("m_fail", a=e["old"]), "fail")
            continue
        f, conf = res
        v = new.to_vaddr(f)
        if v is None:
            e["status"] = "fallo"
            log(t("m_outside", a=e["old"]), "fail")
            continue
        e.update(new=v, new_f=f, conf=conf, status="ok", new_off=new.d[f:f + L])
        # 'on': se conservan los bytes nuevos donde el parche original no tocaba nada
        if len(e["on"]) == L:
            on = bytearray(e["new_off"])
            for i in range(L):
                if e["on"][i] != e["off"][i]:
                    on[i] = e["on"][i]
            e["new_on"] = bytearray(on)
        else:
            e["new_on"] = bytearray(e["on"])
        solved.append((e["old"], v))
        log(f"  0x{e['old']:08X} -> 0x{v:08X}  ({v - e['old']:+#x})  [{t('c_' + conf)}]",
            "ok" if conf == "seguro" else "warn")

    # ---- 2) cuevas de codigo
    used = []
    for e in caves:
        done += 1
        if progress:
            progress(done, total)
        L = len(e["on"])
        f0 = new.to_file(e["old"])
        pos, how = None, ""
        if f0 is not None and f0 + L <= len(new.d) and is_filler(new.d[f0:f0 + max(L, 16)]) \
                and not [u for u in used if e["old"] < u[1] and e["old"] + L > u[0]]:
            pos, how = e["old"], t("how_same")
        else:
            r = find_free(new, L, used)
            if r:
                pos, how = r
        if pos is None:
            e["status"] = "fallo"
            log(t("m_cavefree", a=e["old"]), "fail")
            continue
        f = new.to_file(pos)
        used.append((pos, pos + L))
        e.update(new=pos, conf="probable", status="ok",
                 new_off=(new.d[f:f + L] if f is not None else bytes(L)), new_on=bytearray(e["on"]))
        log(t("m_cave", o=e["old"], n=pos, how=how), "warn")

    # ---- 3) saltos E9/E8 dentro de los parches (hooks y cuevas)
    okents = [e for e in ents.values() if e["status"] == "ok"]
    regions = [(e["old"], e["old"] + len(e["on"]), e["new"]) for e in okents]
    near = [(e["old"], e["new"]) for e in okents if e not in caves]

    def resolve(tg):
        for os_, oe, ns in regions:
            if os_ <= tg <= oe:
                return ns + (tg - os_)
        if near:
            o, n = min(near, key=lambda s: abs(s[0] - tg))
            if abs(o - tg) <= 0x1000:
                return tg + (n - o)
        return None

    for e in okents:
        on = e["on"]
        for i in range(0, len(on) - 4):
            if on[i] in (0xE9, 0xE8):
                (rel,) = struct.unpack_from("<i", on, i + 1)
                tg = e["old"] + i + 5 + rel
                nt = resolve(tg)
                if nt is None:
                    continue
                nr = nt - (e["new"] + i + 5)
                if -0x80000000 <= nr < 0x80000000:
                    e["new_on"][i + 1:i + 5] = struct.pack("<i", nr)
                    log(t("m_jump", a=e["old"], i=i), "info")

    # ---- 4) construir el JSON nuevo (se excluyen los parches con alguna entrada fallida)
    out = copy.deepcopy(patch)
    out["mods"] = []
    kept, dropped = 0, []
    for mod in patch.get("mods", []):
        keys = [key_of(m) for m in mod.get("memory", [])]
        if any(ents[k]["status"] != "ok" for k in keys):
            dropped.append(mod.get("name", "?"))
            continue
        nm = copy.deepcopy(mod)
        for m, k in zip(nm.get("memory", []), keys):
            e = ents[k]
            m["offset"] = f"0x{e['new']:08X}"
            m["off"] = e["new_off"].hex().upper()
            m["on"] = bytes(e["new_on"]).hex().upper()
        out["mods"].append(nm)
        kept += 1
    return out, kept, dropped


# =====================================================================
#  Banner / consola
# =====================================================================
_LETTERS = {
    "I": ["███", " █ ", " █ ", " █ ", "███"],
    "S": [" ████", "█    ", " ███ ", "    █", "████ "],
    "M": ["█   █", "██ ██", "█ █ █", "█   █", "█   █"],
    "A": [" ███ ", "█   █", "█████", "█   █", "█   █"],
    "N": ["█   █", "██  █", "█ █ █", "█  ██", "█   █"],
    "U": ["█   █", "█   █", "█   █", "█   █", " ███ "],
    "E": ["█████", "█    ", "████ ", "█    ", "█████"],
}


def banner_rows():
    return ["  ".join(_LETTERS[c][r] for c in "ISMANUEEE") for r in range(5)]


def print_banner():
    if os.name == "nt":
        os.system("")
    try:
        for row in banner_rows():
            print("\033[38;2;88;101;242m" + row + "\033[0m")
        print("\033[38;2;148;155;164m  " + APP + " · " + t("cli_sub") + "\033[0m\n")
    except UnicodeEncodeError:
        print(f"== {AUTHOR} · {APP} ==\n")


def load_inputs(patch_p, new_p, old_p, log):
    with open(patch_p, "r", encoding="utf-8") as fh:
        patch = json.load(fh)
    with open(new_p, "rb") as fh:
        new = Image(fh.read(), new_p)
    log(t("m_new", name=os.path.basename(new_p), kind=new.kind, size=len(new.d)))
    old = None
    if old_p:
        with open(old_p, "rb") as fh:
            old = Image(fh.read(), old_p)
        log(t("m_old", name=os.path.basename(old_p), kind=old.kind, size=len(old.d)))
    else:
        log(t("m_noold"), "warn")
    return patch, new, old


def run_job(patch_p, new_p, old_p, out_p, ver, log, progress=None):
    patch, new, old = load_inputs(patch_p, new_p, old_p, log)
    out, kept, dropped = port_patch(patch, new, old, log, progress)
    if ver:
        out["version"] = ver
    if not out_p:
        out_p = os.path.splitext(patch_p)[0] + "_portado.json"
    with open(out_p, "w", encoding="utf-8") as fh:
        json.dump(out, fh, indent=4, ensure_ascii=False)
    total = len(patch.get("mods", []))
    log(t("m_saved", p=out_p), "ok")
    log(t("m_ported", k=kept, t=total), "ok" if kept == total else "warn")
    for n in dropped:
        log(t("m_excl", n=n), "fail")
    if kept:
        log(t("m_tip", p=t("c_probable"), d=t("c_dudoso")), "info")
    return out_p


def run_cli(a):
    print_banner()
    colors = {"ok": "\033[92m", "warn": "\033[93m", "fail": "\033[91m", "info": ""}

    def log(m, lv="info"):
        print(colors.get(lv, "") + m + ("\033[0m" if colors.get(lv) else ""))

    run_job(a.patch, a.new, a.old, a.out, a.ver, log)


# =====================================================================
#  Interfaz grafica (estilo Discord)
# =====================================================================
LOGO_B64 = "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"


def run_gui(test_hook=None):
    import tkinter as tk
    from tkinter import filedialog, ttk

    root = tk.Tk()
    root.title(f"{AUTHOR} · {APP}")
    root.configure(bg=BG)
    root.geometry("880x800")
    root.minsize(780, 700)
    try:
        logo = tk.PhotoImage(data=LOGO_B64)
        root.iconphoto(True, logo)
    except tk.TclError:
        logo = None

    # --- cabecera: logo + marca de agua
    head = tk.Frame(root, bg=BG)
    head.pack(fill="x", pady=(14, 2))
    inner = tk.Frame(head, bg=BG)
    inner.pack()
    if logo:
        tk.Label(inner, image=logo, bg=BG, bd=0).pack(side="left", padx=(0, 18))
    titles = tk.Frame(inner, bg=BG)
    titles.pack(side="left")
    tk.Label(titles, text=AUTHOR, bg=BG, fg=BLURPLE, font=("Segoe UI Black", 50, "bold")).pack(anchor="w")
    sub_lbl = tk.Label(titles, bg=BG, fg=MUTED, font=("Segoe UI", 11, "bold"))
    sub_lbl.pack(anchor="w")
    tk.Frame(root, bg=BLURPLE, height=3).pack(fill="x", padx=60, pady=(10, 12))

    bind = []  # (widget, clave, atributo)

    def tr(widget, key, attr="text"):
        bind.append((widget, key, attr))
        widget.configure(**{attr: t(key)})
        return widget

    # --- campos
    panel = tk.Frame(root, bg=PANEL, padx=18, pady=12)
    panel.pack(fill="x", padx=28)
    panel.columnconfigure(0, weight=1)
    v_patch, v_new, v_old, v_ver, v_out = (tk.StringVar() for _ in range(5))

    def pick(var, title_key, save=False, eboot=False):
        types = [("JSON", "*.json")] if not eboot else [("eboot", "*.bin *.elf *.self")]
        types.append((t("ft_all"), "*.*"))
        fn = filedialog.asksaveasfilename if save else filedialog.askopenfilename
        kw = {"defaultextension": ".json"} if save else {}
        f = fn(title=t(title_key), filetypes=types, **kw)
        if f:
            var.set(f)

    def btn(parent, key, cmd, big=False):
        b = tk.Button(parent, command=cmd, bg=BLURPLE, fg="white", activebackground=BLURPLE_H,
                      activeforeground="white", relief="flat", bd=0, cursor="hand2",
                      font=("Segoe UI", 13 if big else 10, "bold"),
                      padx=22 if big else 12, pady=10 if big else 4)
        tr(b, key)
        b.bind("<Enter>", lambda e: b.configure(bg=BLURPLE_H))
        b.bind("<Leave>", lambda e: b.configure(bg=BLURPLE))
        return b

    def row(r, lkey, hkey, var, cmd):
        tr(tk.Label(panel, bg=PANEL, fg=TXT, font=("Segoe UI", 10, "bold")), lkey).grid(row=r * 2, column=0, sticky="w")
        tr(tk.Label(panel, bg=PANEL, fg=MUTED, font=("Segoe UI", 8)), hkey).grid(row=r * 2, column=1, sticky="e", padx=8)
        tk.Entry(panel, textvariable=var, bg=PANEL2, fg=TXT, insertbackground=TXT, relief="flat",
                 font=("Segoe UI", 10), highlightthickness=1, highlightbackground=PANEL2,
                 highlightcolor=BLURPLE).grid(row=r * 2 + 1, column=0, columnspan=2, sticky="ew", pady=(2, 7), ipady=5)
        if cmd:
            btn(panel, "browse", cmd).grid(row=r * 2 + 1, column=2, padx=(8, 0), pady=(2, 7))

    row(0, "l_patch", "h_patch", v_patch, lambda: pick(v_patch, "dlg_patch"))
    row(1, "l_new", "h_new", v_new, lambda: pick(v_new, "dlg_new", eboot=True))
    row(2, "l_old", "h_old", v_old, lambda: pick(v_old, "dlg_old", eboot=True))
    row(3, "l_ver", "h_ver", v_ver, None)
    row(4, "l_out", "h_out", v_out, lambda: pick(v_out, "dlg_save", save=True))

    # --- boton + progreso
    go_btn = btn(root, "go", lambda: start(), big=True)
    go_btn.pack(pady=(12, 6))
    style = ttk.Style()
    style.theme_use("clam")
    style.configure("D.Horizontal.TProgressbar", troughcolor=PANEL, background=BLURPLE,
                    bordercolor=PANEL, lightcolor=BLURPLE, darkcolor=BLURPLE)
    bar = ttk.Progressbar(root, style="D.Horizontal.TProgressbar", mode="determinate")
    bar.pack(fill="x", padx=28, pady=(0, 8))

    # --- log
    box = tk.Frame(root, bg=BG)
    box.pack(fill="both", expand=True, padx=28, pady=(0, 6))
    txt = tk.Text(box, bg="#111214", fg=TXT, relief="flat", font=("Consolas", 10), wrap="word",
                  insertbackground=TXT, height=8)
    sb = tk.Scrollbar(box, command=txt.yview)
    txt.configure(yscrollcommand=sb.set)
    sb.pack(side="right", fill="y")
    txt.pack(side="left", fill="both", expand=True)
    for tag, col in (("ok", GREEN), ("warn", YELLOW), ("fail", RED), ("info", TXT)):
        txt.tag_configure(tag, foreground=col)
    foot = tk.Label(root, bg=BG, fg=MUTED, font=("Segoe UI", 8))
    foot.pack(pady=(0, 8))

    # --- boton circular de idioma (esquina superior derecha)
    corner = tk.Frame(root, bg=BG)
    corner.place(relx=1.0, x=-16, y=12, anchor="ne")
    cv = tk.Canvas(corner, width=48, height=48, bg=BG, highlightthickness=0, cursor="hand2")
    cv.pack()
    code_lbl = tk.Label(corner, bg=BG, fg=MUTED, font=("Segoe UI", 8, "bold"))
    code_lbl.pack()
    disk = cv.create_oval(2, 2, 46, 46, fill=BLURPLE, outline="")
    w = "white"
    cv.create_oval(12, 12, 36, 36, outline=w, width=2)       # globo
    cv.create_oval(19, 12, 29, 36, outline=w, width=2)       # meridiano
    cv.create_line(12, 24, 36, 24, fill=w, width=2)          # ecuador
    cv.create_line(14, 18, 34, 18, fill=w, width=1)          # paralelos
    cv.create_line(14, 30, 34, 30, fill=w, width=1)
    cv.bind("<Enter>", lambda e: cv.itemconfigure(disk, fill=BLURPLE_H))
    cv.bind("<Leave>", lambda e: cv.itemconfigure(disk, fill=BLURPLE))

    def retranslate():
        for wdg, key, attr in bind:
            wdg.configure(**{attr: t(key)})
        sub_lbl.configure(text=t("sub"))
        foot.configure(text=f"{t('footer')} {AUTHOR}")
        code_lbl.configure(text=LANG.upper())

    def set_lang(code):
        global LANG
        LANG = code
        save_lang()
        retranslate()

    def open_menu(event):
        m = tk.Menu(root, tearoff=0, bg=PANEL, fg=TXT, activebackground=BLURPLE,
                    activeforeground="white", bd=0, font=("Segoe UI", 11))
        for code, name in LANGS:
            m.add_command(label=("●  " if code == LANG else "    ") + name,
                          command=lambda c=code: set_lang(c))
        m.tk_popup(event.x_root - 100, event.y_root + 30)

    cv.bind("<Button-1>", open_menu)
    root.set_lang = set_lang
    retranslate()

    q = queue.Queue()

    def log(m, lv="info"):
        q.put(("log", m, lv))

    def progress(i, n):
        q.put(("prog", i, n))

    def poll():
        try:
            while True:
                it = q.get_nowait()
                if it[0] == "log":
                    txt.insert("end", it[1] + "\n", it[2])
                    txt.see("end")
                elif it[0] == "prog":
                    bar.configure(maximum=it[2], value=it[1])
                elif it[0] == "done":
                    go_btn.configure(state="normal", text=t("go"))
        except queue.Empty:
            pass
        root.after(100, poll)

    def work(args):
        try:
            run_job(*args, log=log, progress=progress)
        except Exception as ex:  # noqa: BLE001
            log(t("m_err", e=ex), "fail")
        q.put(("done",))

    def start():
        if not v_patch.get() or not v_new.get():
            log(t("need"), "fail")
            return
        txt.delete("1.0", "end")
        bar.configure(value=0)
        go_btn.configure(state="disabled", text=t("busy"))
        args = (v_patch.get(), v_new.get(), v_old.get() or None, v_out.get() or None, v_ver.get() or None)
        threading.Thread(target=work, args=(args,), daemon=True).start()

    root.start_job = start
    root.vars = (v_patch, v_new, v_old, v_ver, v_out)
    root.log_widget = txt
    poll()
    if test_hook:
        root.after(900, lambda: test_hook(root))
    root.mainloop()


# =====================================================================
def main():
    load_lang()
    ap = argparse.ArgumentParser(description=f"{AUTHOR} · {APP}")
    ap.add_argument("--patch", help="Kyty JSON patch / parche JSON de Kyty")
    ap.add_argument("--new", help="new eboot.bin / eboot.bin nuevo")
    ap.add_argument("--old", help="old eboot.bin (optional, more reliable) / eboot.bin viejo (opcional)")
    ap.add_argument("--out", help="output JSON / JSON de salida")
    ap.add_argument("--ver", help="value for the 'version' field / valor del campo 'version'")
    ap.add_argument("--lang", choices=[c for c, _ in LANGS], help="es | en | pt | ru")
    a = ap.parse_args()
    global LANG
    if a.lang:
        LANG = a.lang
    if a.patch and a.new:
        run_cli(a)
        return
    try:
        run_gui()
    except Exception as ex:  # noqa: BLE001
        print_banner()
        print(t("noui", e=ex))
        ap.print_help()


if __name__ == "__main__":
    main()
