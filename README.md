import sqlite3
import pandas as pd
import yfinance as yf
from datetime import datetime
import warnings

# Ignorar avisos do yfinance para manter o terminal limpo
warnings.filterwarnings('ignore')

def configurar_banco_dados():
    """
    Cria a conexão com o banco de dados local e estrutura a tabela de transações.
    """
    print("⚙️ Estruturando o Banco de Dados...")
    conn = sqlite3.connect('alpha_invest.db')
    cursor = conn.cursor()

    # Criando a tabela de transações
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS tb_transacoes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ticker TEXT NOT NULL,
            tipo TEXT NOT NULL,
            quantidade INTEGER NOT NULL,
            preco_unitario REAL NOT NULL,
            data_operacao TEXT NOT NULL
        )
    ''')

    # Limpando dados antigos para rodar o teste do zero
    cursor.execute('DELETE FROM tb_transacoes')

    # Inserindo um portfólio de exemplo (Ações brasileiras - sufixo .SA)
    aportes = [
        ('WEGE3.SA', 'COMPRA', 100, 32.50, '2023-05-10'),
        ('ITUB4.SA', 'COMPRA', 200, 25.00, '2023-06-15'),
        ('VALE3.SA', 'COMPRA', 50, 65.20, '2023-08-20'),
        ('PETR4.SA', 'COMPRA', 300, 28.90, '2023-11-05')
    ]

    cursor.executemany('''
        INSERT INTO tb_transacoes (ticker, tipo, quantidade, preco_unitario, data_operacao)
        VALUES (?, ?, ?, ?, ?)
    ''', aportes)

    conn.commit()
    return conn

def processar_portfolio(conn):
    """
    Lê os dados do banco, calcula o preço médio e cruza com a cotação atual.
    """
    print("📊 Processando dados e calculando preço médio...")
    # Puxando do banco direto para um DataFrame Pandas
    query = "SELECT ticker, quantidade, preco_unitario FROM tb_transacoes WHERE tipo = 'COMPRA'"
    df_compras = pd.read_sql_query(query, conn)

    # Agrupando para achar a quantidade total e o preço médio pago
    df_portfolio = df_compras.groupby('ticker').apply(
        lambda x: pd.Series({
            'Qtd_Total': x['quantidade'].sum(),
            'Preco_Medio': (x['quantidade'] * x['preco_unitario']).sum() / x['quantidade'].sum()
        })
    ).reset_index()

    return df_portfolio

def buscar_cotacoes_mercado(df_portfolio):
    """
    Bate na API do Yahoo Finance e traz a cotação em tempo real.
    """
    print("🌐 Buscando cotações em tempo real no mercado...")
    tickers = df_portfolio['ticker'].tolist()
    
    # Baixando cotação do último dia útil
    dados_mercado = yf.download(tickers, period="1d", progress=False)['Close']
    
    # Criando um dicionário com os preços atuais
    precos_atuais = {}
    for ticker in tickers:
        # Pega o último valor válido da coluna do ticker
        precos_atuais[ticker] = float(dados_mercado[ticker].iloc[-1])
        
    # Mapeando os preços atuais para o nosso DataFrame
    df_portfolio['Preco_Atual'] = df_portfolio['ticker'].map(precos_atuais)
    return df_portfolio

def gerar_analytics_e_exportar(df_portfolio):
    """
    Calcula rentabilidade, patrimônio e exporta para CSV.
    """
    print("📈 Calculando métricas de performance...")
    
    # Cálculos de Inteligência
    df_portfolio['Total_Investido'] = df_portfolio['Qtd_Total'] * df_portfolio['Preco_Medio']
    df_portfolio['Patrimonio_Atual'] = df_portfolio['Qtd_Total'] * df_portfolio['Preco_Atual']
    df_portfolio['Lucro_Prejuizo'] = df_portfolio['Patrimonio_Atual'] - df_portfolio['Total_Investido']
    df_portfolio['Rentabilidade_%'] = ((df_portfolio['Preco_Atual'] / df_portfolio['Preco_Medio']) - 1) * 100

    # Arredondando valores para ficar bonito no dashboard
    colunas_monetarias = ['Preco_Medio', 'Preco_Atual', 'Total_Investido', 'Patrimonio_Atual', 'Lucro_Prejuizo', 'Rentabilidade_%']
    df_portfolio[colunas_monetarias] = df_portfolio[colunas_monetarias].round(2)

    # Ordenando pelos melhores ativos
    df_portfolio = df_portfolio.sort_values(by='Rentabilidade_%', ascending=False)

    # Exportando a base tratada para CSV
    nome_arquivo = f"dataset_portfolio_{datetime.now().strftime('%Y%m%d')}.csv"
    df_portfolio.to_csv(nome_arquivo, index=False, sep=';', decimal=',')
    
    print(f"\n✅ SUCESSO! Base exportada como: '{nome_arquivo}'")
    print("💡 Dica: Suba esse arquivo no Power BI para criar seus gráficos visuais.\n")
    
    print("Resumo da sua Carteira:")
    print("-" * 50)
    print(df_portfolio[['ticker', 'Rentabilidade_%', 'Lucro_Prejuizo', 'Patrimonio_Atual']].to_string(index=False))


