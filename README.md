"""
Prototipo #1 del código
12/09/2025
"""

import sqlite3
import os
import tkinter as tk
from tkinter import messagebox, simpledialog

DB_FILE = "aivas_gui.db"

# ---------------------------
# Base de datos
# ---------------------------
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
    ]
    c.executemany("INSERT INTO usuarios(nombre,correo,profesor_fav) VALUES(?,?,?)", usuarios)
    c.executemany("INSERT INTO secciones(curso,profesor,horario,capacidad) VALUES(?,?,?,?)", secciones)
    conn.commit()
    conn.close()

# ---------------------------
# Funciones auxiliares
# ---------------------------
def obtener_usuario(correo):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT id,nombre,correo,profesor_fav FROM usuarios WHERE correo=?",(correo,))
    row = c.fetchone()
    conn.close()
    return row

def listar_secciones():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT id,curso,profesor,horario,capacidad FROM secciones")
    rows = c.fetchall()
    conn.close()
    return rows

def inscritos_en_seccion(seccion_id):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("""SELECT u.nombre,u.correo 
                 FROM asignaciones a JOIN usuarios u ON a.usuario_id=u.id
                 WHERE a.seccion_id=?""",(seccion_id,))
    res = c.fetchall()
    conn.close()
    return res

def asignar_usuario(usuario_id,seccion_id):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    try:
        c.execute("INSERT INTO asignaciones(usuario_id,seccion_id) VALUES(?,?)",(usuario_id,seccion_id))
        conn.commit()
        msg="Asignación exitosa."
    except sqlite3.IntegrityError:
        msg="Ya estás asignado a esa sección."
    conn.close()
    return msg

# ---------------------------
# Interfaz gráfica
# ---------------------------
class AIVASApp:
    def __init__(self,root):
        self.root=root
        self.root.title("AIVAS - Prototipo GUI")
        self.usuario=None
        self.build_login()

    def build_login(self):
        for w in self.root.winfo_children():
            w.destroy()
        tk.Label(self.root,text="Login AIVAS",font=("Arial",16)).pack(pady=10)
        tk.Label(self.root,text="Correo institucional (@uvg.edu.gt):").pack()
        self.entry_correo=tk.Entry(self.root,width=40)
        self.entry_correo.pack(pady=5)
        tk.Button(self.root,text="Ingresar",command=self.login).pack()

    def login(self):
        correo=self.entry_correo.get().strip()
        if not correo.endswith("@uvg.edu.gt"):
            messagebox.showerror("Error","Debe usar correo institucional @uvg.edu.gt")
            return
        user=obtener_usuario(correo)
        if not user:
            messagebox.showerror("Error","Usuario no encontrado en BD de prueba.")
            return
        self.usuario=user
        self.build_main()

    def build_main(self):
        for w in self.root.winfo_children():
            w.destroy()
        tk.Label(self.root,text=f"Bienvenido {self.usuario[1]}",font=("Arial",14)).pack(pady=10)
        tk.Button(self.root,text="Ver secciones disponibles",command=self.ver_secciones).pack(fill="x",pady=2)
        tk.Button(self.root,text="Ver inscritos en sección",command=self.ver_inscritos).pack(fill="x",pady=2)
        tk.Button(self.root,text="Asignarme a sección",command=self.asignarme).pack(fill="x",pady=2)
        tk.Button(self.root,text="Cerrar sesión",command=self.build_login).pack(fill="x",pady=2)

    def ver_secciones(self):
        win=tk.Toplevel(self.root)
        win.title("Secciones disponibles")
        lista=tk.Listbox(win,width=80)
        lista.pack()
        for sid,curso,prof,horario,cap in listar_secciones():
            lista.insert(tk.END,f"{sid}) {curso} - {prof} - {horario} (Cap: {cap})")

    def ver_inscritos(self):
        sid=simpledialog.askinteger("Inscritos","ID de sección:")
        if not sid:
            return
        inscritos=inscritos_en_seccion(sid)
        if not inscritos:
            messagebox.showinfo("Inscritos","No hay estudiantes.")
            return
        win=tk.Toplevel(self.root)
        win.title("Inscritos")
        lista=tk.Listbox(win,width=60)
        lista.pack()
        for nom,cor in inscritos:
            lista.insert(tk.END,f"{nom} <{cor}>")

    def asignarme(self):
        sid=simpledialog.askinteger("Asignación","ID de sección a inscribirse:")
        if not sid:
            return
        msg=asignar_usuario(self.usuario[0],sid)
        messagebox.showinfo("Resultado",msg)

# ---------------------------
# Main
# ---------------------------
if __name__=="__main__":
    init_db(force=False)
    seed_data()
    root=tk.Tk()
    app=AIVASApp(root)
    root.mainloop()


