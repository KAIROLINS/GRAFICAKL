from flask import Flask, render_template, request, jsonify, redirect, url_for
import sqlite3

app = Flask(__name__)

# CONFIGURAÇÃO E CRIAÇÃO DO BANCO DE DADOS
def init_db():
    conn = sqlite3.connect('grafica.db')
    cursor = conn.cursor()
    
    # Tabela de Estoque
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS estoque (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            nome TEXT NOT NULL,
            quantidade INTEGER NOT NULL,
            unidade TEXT NOT NULL
        )
    ''')
    
    # Tabela de Financeiro (Fluxo de Caixa)
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS financeiro (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            descricao TEXT NOT NULL,
            tipo TEXT NOT NULL,
            valor REAL NOT NULL,
            data TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Inserir dados de teste iniciais se as tabelas estiverem vazias
    cursor.execute("SELECT COUNT(*) FROM estoque")
    if cursor.fetchone()[0] == 0:
        cursor.execute("INSERT INTO estoque (nome, quantidade, unidade) VALUES ('Papel Sulfite A4 75g', 120, 'folhas')")
        cursor.execute("INSERT INTO estoque (nome, quantidade, unidade) VALUES ('Papel Couchê 150g A4', 45, 'folhas')")
        
    cursor.execute("SELECT COUNT(*) FROM financeiro")
    if cursor.fetchone()[0] == 0:
        cursor.execute("INSERT INTO financeiro (descricao, tipo, valor) VALUES ('Serviço Cartões de Visita', 'entrada', 150.00)")
        cursor.execute("INSERT INTO financeiro (descricao, tipo, valor) VALUES ('Compra de Tintas', 'saida', 65.00)")
        
    conn.commit()
    conn.close()

# ROTAS DO SISTEMA
@app.route('/')
def index():
    conn = sqlite3.connect('grafica.db')
    cursor = conn.cursor()
    
    # Buscar Estoque
    cursor.execute("SELECT * FROM estoque")
    estoque = [dict(id=row[0], nome=row[1], quantidade=row[2], unidade=row[3]) for row in cursor.fetchall()]
    
    # Buscar Financeiro
    cursor.execute("SELECT * FROM financeiro ORDER BY id DESC")
    financeiro = [dict(id=row[0], descricao=row[1], tipo=row[2], valor=row[3], data=row[4]) for row in cursor.fetchall()]
    
    # Calcular KPIs do Dashboard
    cursor.execute("SELECT SUM(valor) FROM financeiro WHERE tipo='entrada'")
    entradas = cursor.fetchone()[0] or 0.0
    
    cursor.execute("SELECT SUM(valor) FROM financeiro WHERE tipo='saida'")
    saidas = cursor.fetchone()[0] or 0.0
    
    saldo = entradas - saidas
    
    conn.close()
    
    return render_template('index.html', estoque=estoque, financeiro=financeiro, entradas=entradas, saidas=saidas, saldo=saldo)

# API/ROTA PARA ADICIONAR INSUMO NO ESTOQUE
@app.route('/estoque/adicionar', methods=['POST'])
def adicionar_insumo():
    nome = request.form.get('nome')
    quantidade = int(request.form.get('quantidade', 0))
    unidade = request.form.get('unidade')
    
    conn = sqlite3.connect('grafica.db')
    cursor = conn.cursor()
    cursor.execute("INSERT INTO estoque (nome, quantidade, unidade) VALUES (?, ?, ?)", (nome, quantidade, unidade))
    conn.commit()
    conn.close()
    return redirect(url_for('index') + '#estoque')

# API/ROTA PARA ADICIONAR TRANSAÇÃO FINANCEIRA
@app.route('/financeiro/adicionar', methods=['POST'])
def adicionar_transacao():
    descricao = request.form.get('descricao')
    valor = float(request.form.get('valor', 0.0))
    tipo = request.form.get('tipo')
    
    conn = sqlite3.connect('grafica.db')
    cursor = conn.cursor()
    cursor.execute("INSERT INTO financeiro (descricao, tipo, valor) VALUES (?, ?, ?)", (descricao, tipo, valor))
    conn.commit()
    conn.close()
    return redirect(url_for('index') + '#financeiro')

# API/ROTA PARA ALTERAR QUANTIDADE DE ESTOQUE (+ ou -) VIA AJAX
@app.route('/estoque/alterar/<int:id>/<int:valor>', methods=['POST'])
def alterar_qtd(id, valor):
    conn = sqlite3.connect('grafica.db')
    cursor = conn.cursor()
    cursor.execute("UPDATE estoque SET quantidade = MAX(0, quantidade + ?) WHERE id = ?", (valor, id))
    conn.commit()
    conn.close()
    return jsonify({"success": True})

if __name__ == '__main__':
    init_db()
    app.run(debug=True, port=5000)