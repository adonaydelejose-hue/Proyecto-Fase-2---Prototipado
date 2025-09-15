import sqlite3, os
import tkinter as tk
from tkinter import ttk, messagebox, simpledialog
from datetime import time

DB_FILE = "aivas_gui_final.db"

# ------------------ Utilidades de horario ------------------
def parse_schedule(s):
    try:
        day, start_s, end_s = s.split(",")
        h1, m1 = map(int, start_s.split(":"))
        h2, m2 = map(int, end_s.split(":"))
        return (day, time(h1, m1), time(h2, m2))
    except Exception:
        return ("", time(0,0), time(0,0))

def overlap(a, b):
    d1, s1, e1 = a
    d2, s2, e2 = b
    if d1 != d2:
        return False
    return max(s1, s2) < min(e1, e2)

# ------------------ DB: init + seed ------------------
def init_db(force=False):
    if force and os.path.exists(DB_FILE):
        os.remove(DB_FILE)
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("""CREATE TABLE IF NOT EXISTS usuarios(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nombre TEXT,
        correo TEXT UNIQUE,
        profesor_fav TEXT
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS secciones(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        curso TEXT,
        profesor TEXT,
        horario TEXT,
        capacidad INTEGER
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS asignaciones(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        usuario_id INTEGER,
        seccion_id INTEGER,
        UNIQUE(usuario_id,seccion_id)
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS amigos(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        usuario_id INTEGER,
        amigo_id INTEGER,
        UNIQUE(usuario_id,amigo_id)
    )""")
    conn.commit()
    conn.close()

def seed_data():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT COUNT(*) FROM usuarios")
    if c.fetchone()[0] > 0:
        conn.close()
        return
    usuarios = [
        ("Ana López","ana@uvg.edu.gt","Dr. Ramírez"),
        ("Luis Pérez","luis@uvg.edu.gt","Dra. Gómez"),
        ("María Ruiz","maria@uvg.edu.gt","Dr. Ramírez"),
        ("Carlos Mejía","carlos@uvg.edu.gt","Dra. Gómez"),
        ("José León","jose@uvg.edu.gt","Dr. Rivera"),
        ("Juan Morales","juan@uvg.edu.gt","Dr. Ramírez"),
        ("Sofía Torres","sofia@uvg.edu.gt","Dr. Rivera"),
        ("Paola Cabrera","paola@uvg.edu.gt","Dr. Hernández"),
    ]
    secciones = [
        ("Cálculo I","Dr. Ramírez","Lun,08:00,10:00", 34),
        ("Cálculo I","Dra. Gómez","Mar,10:00,12:00", 34),
        ("Cálculo I","Dr. Rivera","Mie,08:00,10:00", 34),
        ("Programación Básica","Dr. Rivera","Lun,10:00,12:00", 30),
        ("Programación Básica","Dra. Gómez","Mar,08:00,10:00", 30),
        ("Física I","Dr. Hernández","Mie,10:00,12:00", 32),
        ("Física I","Dr. Ramírez","Jue,08:00,10:00", 32),
        ("Cálculo II","Dra. Gómez","Vie,08:00,10:00", 34),
        ("Álgebra Lineal","Dr. Rivera","Jue,10:00,12:00", 30),
    ]
    c.executemany("INSERT INTO usuarios(nombre,correo,profesor_fav) VALUES(?,?,?)", usuarios)
    c.executemany("INSERT INTO secciones(curso,profesor,horario,capacidad) VALUES(?,?,?,?)", secciones)
    conn.commit()

    # Asignaciones de ejemplo
    def sid(curso, prof):
        c.execute("SELECT id FROM secciones WHERE curso=? AND profesor=?", (curso, prof))
        r = c.fetchone()
        return r[0] if r else None

    ids = {}
    for correo in [u[1] for u in usuarios]:
        c.execute("SELECT id FROM usuarios WHERE correo=?", (correo,))
        ids[correo] = c.fetchone()[0]

    asign = [
        (ids["ana@uvg.edu.gt"], sid("Cálculo I","Dr. Ramírez")),
        (ids["maria@uvg.edu.gt"], sid("Cálculo I","Dr. Ramírez")),
        (ids["juan@uvg.edu.gt"], sid("Cálculo I","Dr. Ramírez")),
        (ids["luis@uvg.edu.gt"], sid("Cálculo I","Dra. Gómez")),
        (ids["carlos@uvg.edu.gt"], sid("Programación Básica","Dr. Rivera")),
    ]
    for u, s in asign:
        if s:
            try: c.execute("INSERT INTO asignaciones(usuario_id,seccion_id) VALUES(?,?)",(u,s))
            except sqlite3.IntegrityError: pass

    conn.commit()
    conn.close()

# ------------------ Data helpers ------------------
def get_conn(): return sqlite3.connect(DB_FILE)

def normalize_correo(raw: str) -> str:
    raw = raw.strip()
    if "@" not in raw:
        raw = raw + "@uvg.edu.gt"
    return raw.lower()

def inferir_nombre(correo: str) -> str:
    local = correo.split("@")[0]
    for sep in [".","_","-"]:
        local = local.replace(sep, " ")
    partes = [p for p in local.split() if p]
    return " ".join(p.capitalize() for p in partes) if partes else "Usuario UVG"

def obtener_usuario(correo):
    conn = get_conn(); c = conn.cursor()
    c.execute("SELECT id,nombre,correo,profesor_fav FROM usuarios WHERE correo=?", (correo,))
    row = c.fetchone()
    conn.close()
    return row

def crear_usuario_si_no_existe(correo):
    u = obtener_usuario(correo)
    if u: return u
    if not correo.endswith("@uvg.edu.gt"): return None
    conn = get_conn(); c = conn.cursor()
    try:
        c.execute("INSERT INTO usuarios(nombre,correo,profesor_fav) VALUES(?,?,?)",
                  (inferir_nombre(correo), correo, None))
        conn.commit()
    except sqlite3.IntegrityError:
        pass
    conn.close()
    return obtener_usuario(correo)

def listar_secciones():
    conn = get_conn(); c = conn.cursor()
    c.execute("SELECT id,curso,profesor,horario,capacidad FROM secciones")
    rows = c.fetchall(); conn.close()
    return rows

def secciones_por_curso(curso):
    conn = get_conn(); c = conn.cursor()
    c.execute("SELECT id,curso,profesor,horario,capacidad FROM secciones WHERE curso=?", (curso,))
    rows = c.fetchall(); conn.close()
    return rows

def cupo_disponible(seccion_id):
    conn = get_conn(); c = conn.cursor()
    c.execute("SELECT capacidad FROM secciones WHERE id=?", (seccion_id,))
    r = c.fetchone()
    if not r:
        conn.close(); return None
    capacidad = r[0]
    c.execute("SELECT COUNT(*) FROM asignaciones WHERE seccion_id=?", (seccion_id,))
    cur = c.fetchone()[0]
    conn.close()
    return capacidad - cur

def inscritos_en_seccion(seccion_id):
    conn = get_conn(); c = conn.cursor()
    c.execute("""SELECT u.id,u.nombre,u.correo
                 FROM asignaciones a JOIN usuarios u ON a.usuario_id=u.id
                 WHERE a.seccion_id=?""", (seccion_id,))
    rows = c.fetchall(); conn.close()
    return rows

def mis_secciones(usuario_id):
    conn = get_conn(); c = conn.cursor()
    c.execute("""SELECT s.id,s.curso,s.profesor,s.horario
                 FROM asignaciones a JOIN secciones s ON a.seccion_id=s.id
                 WHERE a.usuario_id=?""", (usuario_id,))
    rows = c.fetchall(); conn.close()
    return rows

def curso_de_seccion(seccion_id):
    conn = get_conn(); c = conn.cursor()
    c.execute("SELECT curso FROM secciones WHERE id=?", (seccion_id,))
    r = c.fetchone(); conn.close()
    return r[0] if r else None

def horario_de_seccion(seccion_id):
    conn = get_conn(); c = conn.cursor()
    c.execute("SELECT horario FROM secciones WHERE id=?", (seccion_id,))
    r = c.fetchone(); conn.close()
    return parse_schedule(r[0]) if r else None

def asignar(usuario_id, seccion_id):
    # Valida existencia
    if cupo_disponible(seccion_id) is None:
        return False, "La sección no existe."

    # Bloqueo por curso duplicado
    curso_obj = curso_de_seccion(seccion_id)
    for sid, curso, prof, hor in mis_secciones(usuario_id):
        if curso == curso_obj:
            return False, f"Ya estás asignado a '{curso_obj}' en otra sección."

    # Bloqueo por choque horario (decir con cuál)
    nuevo = horario_de_seccion(seccion_id)
    for sid, curso, prof, hor in mis_secciones(usuario_id):
        if overlap(parse_schedule(hor), nuevo):
            return False, f"Choque de horario con {curso} ({hor})."

    # Bloqueo por cupo
    disp = cupo_disponible(seccion_id)
    if disp is not None and disp <= 0:
        return False, "No hay cupo disponible."

    # Insertar
    conn = get_conn(); c = conn.cursor()
    try:
        c.execute("INSERT INTO asignaciones(usuario_id,seccion_id) VALUES(?,?)", (usuario_id, seccion_id))
        conn.commit()
    except sqlite3.IntegrityError:
        conn.close()
        return False, "Ya estás asignado a esa sección."
    conn.close()
    return True, "Asignación exitosa."

def retirar(usuario_id, seccion_id):
    conn = get_conn(); c = conn.cursor()
    c.execute("DELETE FROM asignaciones WHERE usuario_id=? AND seccion_id=?", (usuario_id, seccion_id))
    conn.commit(); conn.close()
    return True, "Te retiraste de la sección."

# ------------------ GUI ------------------
class AIVASApp:
    def __init__(self, root):
        self.root = root
        self.root.title("AIVAS – UVG")
        self.usuario = None
        self._build_login()

    # -------- Login --------
    def _build_login(self):
        for w in self.root.winfo_children(): w.destroy()
        frame = ttk.Frame(self.root, padding=14); frame.pack(fill="both", expand=True)
        ttk.Label(frame, text="Login AIVAS", font=("Arial",16,"bold")).pack(pady=(0,10))
        ttk.Label(frame, text="Correo institucional (@uvg.edu.gt):").pack()
        self.entry_correo = ttk.Entry(frame, width=42); self.entry_correo.pack(pady=6)
        ttk.Button(frame, text="Ingresar", command=self._login).pack()
        ttk.Separator(frame).pack(fill="x", pady=10)
        ttk.Button(frame, text="Reset seed (recrear datos de prueba)", command=self._reset_seed).pack()

    def _login(self):
        correo = normalize_correo(self.entry_correo.get())
        if not correo.endswith("@uvg.edu.gt"):
            messagebox.showerror("Correo inválido", "Use su correo institucional @uvg.edu.gt")
            return
        user = crear_usuario_si_no_existe(correo)
        if not user:
            messagebox.showerror("Error", "No se pudo crear o recuperar el usuario.")
            return
        self.usuario = user
        self._build_main()

    def _reset_seed(self):
        if messagebox.askyesno("Reset", "Esto borrará y recreará los datos de prueba. ¿Continuar?"):
            init_db(force=True); seed_data()
            messagebox.showinfo("Listo", "Datos reiniciados.")

    # -------- Main --------
    def _build_main(self):
        for w in self.root.winfo_children(): w.destroy()
        container = ttk.Frame(self.root, padding=10); container.pack(fill="both", expand=True)
        header = ttk.Frame(container); header.pack(fill="x")
        ttk.Label(header, text=f"Bienvenido/a, {self.usuario[1]}", font=("Arial",14,"bold")).pack(side="left")

        # Barra de búsqueda
        searchbar = ttk.Frame(container); searchbar.pack(fill="x", pady=(8,6))
        ttk.Label(searchbar, text="Buscar (curso o profesor):").pack(side="left")
        self.var_q = tk.StringVar()
        ent = ttk.Entry(searchbar, textvariable=self.var_q, width=40)
        ent.pack(side="left", padx=6)
        ttk.Button(searchbar, text="Buscar", command=self._reload_table).pack(side="left")
        ttk.Button(searchbar, text="Limpiar", command=self._clear_search).pack(side="left", padx=4)

        # Tabla secciones
        table_frame = ttk.Frame(container); table_frame.pack(fill="both", expand=True)
        cols = ("id","curso","profesor","horario","cupo")
        self.tree = ttk.Treeview(table_frame, columns=cols, show="headings", height=12)
        for c in cols:
            self.tree.heading(c, text=c.capitalize())
            self.tree.column(c, width=120 if c!="horario" else 200, anchor="center")
        self.tree.pack(side="left", fill="both", expand=True)
        sb = ttk.Scrollbar(table_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscroll=sb.set); sb.pack(side="right", fill="y")

        self._reload_table()

        hint = ttk.Label(container, text="Doble clic en una fila para ver inscritos / asignarte.", foreground="#555")
        hint.pack(anchor="w", pady=(4,8))

        # Acciones
        actions = ttk.Frame(container); actions.pack(fill="x")
        ttk.Button(actions, text="Ver inscritos", command=self._ver_inscritos_dialog).pack(side="left")
        ttk.Button(actions, text="Asignarme", command=self._asignarme_dialog).pack(side="left", padx=6)
        ttk.Button(actions, text="Retirarme", command=self._retirarme_dialog).pack(side="left")
        ttk.Button(actions, text="Mis secciones", command=self._mis_secciones).pack(side="left", padx=12)
        ttk.Button(actions, text="Por curso → secciones", command=self._curso_a_secciones).pack(side="left")
        ttk.Button(actions, text="Cerrar sesión", command=self._build_login).pack(side="right")

        # Doble clic
        self.tree.bind("<Double-1>", self._on_dclick)

    def _clear_search(self):
        self.var_q.set("")
        self._reload_table()

    def _reload_table(self):
        for i in self.tree.get_children():
            self.tree.delete(i)
        q = self.var_q.get().strip().lower()
        rows = listar_secciones()
        for sid, curso, prof, hor, cap in rows:
            if q and (q not in curso.lower() and q not in prof.lower()):
                continue
            cd = cupo_disponible(sid)
            self.tree.insert("", "end", values=(sid, curso, prof, hor, cd if cd is not None else "-"))

    def _selected_seccion_id(self):
        sel = self.tree.selection()
        if not sel:
            messagebox.showwarning("Selecciona", "Elegí una sección de la tabla.")
            return None
        vals = self.tree.item(sel[0], "values")
        return int(vals[0])

    def _on_dclick(self, _):
        sid = self._selected_seccion_id()
        if sid is None: return
        self._inscritos_popup(sid)

    # -------- Ver inscritos --------
    def _ver_inscritos_dialog(self):
        sid = self._selected_seccion_id()
        if sid is None: return
        self._inscritos_popup(sid)

    def _inscritos_popup(self, seccion_id):
        win = tk.Toplevel(self.root); win.title(f"Inscritos · Sección {seccion_id}")
        lst = tk.Listbox(win, width=70, height=14); lst.pack(padx=8, pady=8)
        data = inscritos_en_seccion(seccion_id)
        if not data:
            lst.insert(tk.END, "No hay estudiantes inscritos.")
            return
        for uid, nom, cor in data:
            lst.insert(tk.END, f"{nom} <{cor}>")
        ttk.Label(win, text=f"Total: {len(data)}").pack(pady=(0,6))

    # -------- Asignarme --------
    def _asignarme_dialog(self):
        sid = self._selected_seccion_id()
        if sid is None: return
        # Confirmación
        vals = self.tree.item(self.tree.selection()[0], "values")
        curso, prof, hor = vals[1], vals[2], vals[3]
        if not messagebox.askyesno("Confirmar",
            f"¿Asignarte a:\n{curso} · {prof} · {hor}?"):
            return
        ok, msg = asignar(self.usuario[0], sid)
        if ok:
            messagebox.showinfo("Asignación", msg)
            self._reload_table()   # refresca cupo
        else:
            messagebox.showerror("No se pudo asignar", msg)

    # -------- Retirarme --------
    def _retirarme_dialog(self):
        # elige entre mis secciones
        mis = mis_secciones(self.usuario[0])
        if not mis:
            messagebox.showinfo("Retiro", "No estás asignado a ninguna sección.")
            return
        opciones = [f"{sid} · {curso} · {prof} · {hor}" for sid, curso, prof, hor in mis]
        choice = simpledialog.askstring("Retirarme", "Copia/pega una opción exacta:\n" + "\n".join(opciones))
        if not choice: return
        try:
            sid = int(choice.split(" · ")[0])
        except:
            messagebox.showerror("Formato inválido", "Selecciona una opción válida.")
            return
        if not messagebox.askyesno("Confirmar", "¿Seguro que deseas retirarte?"):
            return
        ok, msg = retirar(self.usuario[0], sid)
        if ok:
            messagebox.showinfo("Retiro", msg)
            self._reload_table()
        else:
            messagebox.showerror("Retiro", msg)

    # -------- Mis secciones --------
    def _mis_secciones(self):
        win = tk.Toplevel(self.root); win.title("Mis secciones")
        lst = tk.Listbox(win, width=75, height=10); lst.pack(padx=8, pady=8)
        data = mis_secciones(self.usuario[0])
        if not data:
            lst.insert(tk.END, "No estás asignado a ninguna sección.")
            return
        for sid, curso, prof, hor in data:
            lst.insert(tk.END, f"{sid} · {curso} · {prof} · {hor}")

    # -------- Curso → Secciones --------
    def _curso_a_secciones(self):
        # selector de curso
        cursos = sorted({r[1] for r in listar_secciones()})
        win = tk.Toplevel(self.root); win.title("Seleccionar curso")
        frm = ttk.Frame(win, padding=10); frm.pack(fill="both", expand=True)
        ttk.Label(frm, text="Curso:").grid(row=0, column=0, sticky="w")
        cb = ttk.Combobox(frm, values=cursos, state="readonly", width=40)
        cb.grid(row=0, column=1, padx=6, pady=6)

        cols = ("id","curso","profesor","horario","cupo")
        tree = ttk.Treeview(frm, columns=cols, show="headings", height=8)
        for c in cols:
            tree.heading(c, text=c.capitalize())
            tree.column(c, width=120 if c!="horario" else 200, anchor="center")
        tree.grid(row=1, column=0, columnspan=3, sticky="nsew")
        frm.grid_rowconfigure(1, weight=1); frm.grid_columnconfigure(2, weight=1)

        def cargar():
            for i in tree.get_children(): tree.delete(i)
            curso = cb.get()
            if not curso: return
            for sid, c, p, h, cap in secciones_por_curso(curso):
                tree.insert("", "end", values=(sid, c, p, h, cupo_disponible(sid)))
        ttk.Button(frm, text="Ver secciones", command=cargar).grid(row=0, column=2, padx=6)

        def asignar_dbl(_):
            sel = tree.selection()
            if not sel: return
            vals = tree.item(sel[0], "values")
            sid = int(vals[0])
            curso, prof, hor = vals[1], vals[2], vals[3]
            if not messagebox.askyesno("Confirmar", f"¿Asignarte a:\n{curso} · {prof} · {hor}?"):
                return
            ok, msg = asignar(self.usuario[0], sid)
            if ok:
                messagebox.showinfo("Asignación", msg)
                cargar()
            else:
                messagebox.showerror("No se pudo asignar", msg)

        tree.bind("<Double-1>", asignar_dbl)

# ------------------ Main ------------------
if __name__ == "__main__":
    init_db(force=False)
    seed_data()
    root = tk.Tk()
    try:
        style = ttk.Style()
        if "clam" in style.theme_names():
            style.theme_use("clam")
    except Exception:
        pass
    AIVASApp(root)
    root.mainloop()

