# ESTOQUE-E-PRODU-O

## 🧩 MÓDULO 3 — ESTOQUE E PRODUÇÃO
🎯 Objetivo

Controlar produtos, entradas/saídas de estoque e custos médios, integrando-se aos módulos:

Vendas → Saída de produtos.

Compras (futuro) → Entrada de produtos.

Finanças → Valorizar custos de estoque e movimentações.

📁 1️⃣ Estrutura de Pastas

Backend:

modules/
└── estoque/
    ├── produto.model.js
    ├── movimento.model.js
    ├── estoque.controller.js
    └── estoque.routes.js


Frontend:

pages/
└── Estoque/
    ├── ProdutosList.jsx
    ├── ProdutoForm.jsx
    ├── MovimentacoesList.jsx
    └── MovimentoForm.jsx

🧠 2️⃣ Backend — Estrutura principal
📌 produto.model.js
import db from '../../database/connection.js';
import { DataTypes } from 'sequelize';

const Produto = db.define('Produto', {
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  nome: { type: DataTypes.STRING(100), allowNull: false },
  descricao: { type: DataTypes.STRING(255) },
  preco_custo: { type: DataTypes.DECIMAL(10,2), allowNull: false },
  preco_venda: { type: DataTypes.DECIMAL(10,2), allowNull: false },
  quantidade: { type: DataTypes.INTEGER, defaultValue: 0 },
  unidade: { type: DataTypes.STRING(10), defaultValue: 'un' },
});

export default Produto;

📌 movimento.model.js
import db from '../../database/connection.js';
import { DataTypes } from 'sequelize';
import Produto from './produto.model.js';

const Movimento = db.define('Movimento', {
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  produto_id: { type: DataTypes.INTEGER, allowNull: false },
  tipo: { type: DataTypes.ENUM('entrada', 'saida'), allowNull: false },
  quantidade: { type: DataTypes.INTEGER, allowNull: false },
  data: { type: DataTypes.DATE, defaultValue: DataTypes.NOW },
  observacao: { type: DataTypes.STRING(255) },
});

Movimento.belongsTo(Produto, { foreignKey: 'produto_id' });

export default Movimento;

📌 estoque.controller.js
import Produto from './produto.model.js';
import Movimento from './movimento.model.js';

// ---------- PRODUTOS ----------
export async function listarProdutos(req, res) {
  const produtos = await Produto.findAll();
  res.json(produtos);
}

export async function criarProduto(req, res) {
  const produto = await Produto.create(req.body);
  res.status(201).json(produto);
}

export async function atualizarProduto(req, res) {
  const produto = await Produto.findByPk(req.params.id);
  if (!produto) return res.status(404).json({ error: 'Produto não encontrado' });
  await produto.update(req.body);
  res.json(produto);
}

export async function excluirProduto(req, res) {
  const produto = await Produto.findByPk(req.params.id);
  if (!produto) return res.status(404).json({ error: 'Produto não encontrado' });
  await produto.destroy();
  res.json({ message: 'Produto excluído com sucesso' });
}

// ---------- MOVIMENTOS ----------
export async function registrarMovimento(req, res) {
  const { produto_id, tipo, quantidade, observacao } = req.body;
  const produto = await Produto.findByPk(produto_id);
  if (!produto) return res.status(404).json({ error: 'Produto não encontrado' });

  // Atualiza estoque
  if (tipo === 'entrada') produto.quantidade += quantidade;
  if (tipo === 'saida') produto.quantidade -= quantidade;
  await produto.save();

  // Cria movimento
  const movimento = await Movimento.create({ produto_id, tipo, quantidade, observacao });
  res.status(201).json(movimento);
}

export async function listarMovimentos(req, res) {
  const movimentos = await Movimento.findAll({ include: Produto });
  res.json(movimentos);
}

📌 estoque.routes.js
import express from 'express';
import {
  listarProdutos,
  criarProduto,
  atualizarProduto,
  excluirProduto,
  registrarMovimento,
  listarMovimentos
} from './estoque.controller.js';

const router = express.Router();

// Produtos
router.get('/produtos', listarProdutos);
router.post('/produtos', criarProduto);
router.put('/produtos/:id', atualizarProduto);
router.delete('/produtos/:id', excluirProduto);

// Movimentações
router.get('/movimentos', listarMovimentos);
router.post('/movimentos', registrarMovimento);

export default router;


E no seu main.ts (ou server.ts):

import estoqueRoutes from './modules/estoque/estoque.routes.js';
app.use('/api', estoqueRoutes);

💾 3️⃣ Banco de Dados
CREATE TABLE produtos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  nome VARCHAR(100) NOT NULL,
  descricao VARCHAR(255),
  preco_custo DECIMAL(10,2) NOT NULL,
  preco_venda DECIMAL(10,2) NOT NULL,
  quantidade INT DEFAULT 0,
  unidade VARCHAR(10) DEFAULT 'un'
);

CREATE TABLE movimentos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  produto_id INT NOT NULL,
  tipo ENUM('entrada','saida') NOT NULL,
  quantidade INT NOT NULL,
  data DATETIME DEFAULT CURRENT_TIMESTAMP,
  observacao VARCHAR(255),
  FOREIGN KEY (produto_id) REFERENCES produtos(id)
);

⚙️ 4️⃣ Frontend — React
📌 ProdutosList.jsx
import { useEffect, useState } from 'react';
import axios from 'axios';

export default function ProdutosList() {
  const [produtos, setProdutos] = useState([]);

  useEffect(() => {
    axios.get('http://localhost:3001/api/produtos')
      .then(res => setProdutos(res.data));
  }, []);

  return (
    <div className="p-4">
      <h2 className="text-xl font-semibold mb-3">Produtos em Estoque</h2>
      <table className="w-full border">
        <thead>
          <tr>
            <th>Nome</th>
            <th>Qtd</th>
            <th>Preço Venda</th>
          </tr>
        </thead>
        <tbody>
          {produtos.map(p => (
            <tr key={p.id}>
              <td>{p.nome}</td>
              <td>{p.quantidade}</td>
              <td>R$ {p.preco_venda}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

📌 ProdutoForm.jsx
import { useState } from 'react';
import axios from 'axios';

export default function ProdutoForm() {
  const [form, setForm] = useState({
    nome: '', descricao: '', preco_custo: '', preco_venda: '', quantidade: ''
  });

  const handleChange = e => setForm({ ...form, [e.target.name]: e.target.value });

  const handleSubmit = async e => {
    e.preventDefault();
    await axios.post('http://localhost:3001/api/produtos', form);
    alert('Produto cadastrado!');
    setForm({ nome: '', descricao: '', preco_custo: '', preco_venda: '', quantidade: '' });
  };

  return (
    <form onSubmit={handleSubmit} className="p-4 bg-gray-100 rounded">
      <h2 className="text-lg font-semibold mb-3">Cadastrar Produto</h2>
      <input name="nome" placeholder="Nome" value={form.nome} onChange={handleChange} className="block mb-2 p-1 border" />
      <input name="descricao" placeholder="Descrição" value={form.descricao} onChange={handleChange} className="block mb-2 p-1 border" />
      <input name="preco_custo" type="number" step="0.01" placeholder="Preço de Custo" value={form.preco_custo} onChange={handleChange} className="block mb-2 p-1 border" />
      <input name="preco_venda" type="number" step="0.01" placeholder="Preço de Venda" value={form.preco_venda} onChange={handleChange} className="block mb-2 p-1 border" />
      <input name="quantidade" type="number" placeholder="Quantidade" value={form.quantidade} onChange={handleChange} className="block mb-2 p-1 border" />
      <button className="bg-blue-600 text-white px-3 py-1 rounded">Salvar</button>
    </form>
  );
}

📌 MovimentacoesList.jsx
import { useEffect, useState } from 'react';
import axios from 'axios';

export default function MovimentacoesList() {
  const [movimentos, setMovimentos] = useState([]);

  useEffect(() => {
    axios.get('http://localhost:3001/api/movimentos')
      .then(res => setMovimentos(res.data));
  }, []);

  return (
    <div className="p-4">
      <h2 className="text-lg font-semibold mb-3">Movimentações de Estoque</h2>
      <table className="w-full border">
        <thead>
          <tr>
            <th>Produto</th>
            <th>Tipo</th>
            <th>Quantidade</th>
            <th>Data</th>
          </tr>
        </thead>
        <tbody>
          {movimentos.map(m => (
            <tr key={m.id}>
              <td>{m.Produto?.nome}</td>
              <td>{m.tipo}</td>
              <td>{m.quantidade}</td>
              <td>{new Date(m.data).toLocaleString()}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

🔗 5️⃣ Integrações futuras
Módulo	Integração	Descrição
Vendas	Saída de produtos	Cada venda gera uma movimentação de tipo “saida”.
Compras	Entrada de produtos	Entrada de notas ou pedidos de compra.
Financeiro	Custo de produtos	Valor do custo pode impactar o fluxo de caixa.
🧭 6️⃣ Resumo

✅ Controle completo de produtos e movimentações
✅ Backend com Sequelize e relacionamentos
✅ Frontend funcional (listagem e cadastro)
✅ Preparado para integrar com Vendas e Finanças
