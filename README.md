# bot.py
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
import pandas as pd
import io
import matplotlib.pyplot as plt
from fpdf import FPDF
import os
import random
import textwrap

# ================================
# LER VARIÁVEIS DE AMBIENTE
# ================================
TOKEN = os.getenv("TELEGRAM_TOKEN")  # configure no Render
# Opcional: chave para Gemini/API de terceiros. Se deixar vazio, o bot usará resumo local gratuito.
GEMINI_KEY = os.getenv("GEMINI_KEY", "")

if not TOKEN:
    raise Exception("Erro: variável de ambiente TELEGRAM_TOKEN não encontrada.")

# ================================
# FUNÇÕES DE INSIGHT (GRATUITAS)
# ================================
def generate_local_insights(df: pd.DataFrame) -> str:
    """Gera um texto de insights simples com base nas estatísticas do DataFrame (sem usar API externa)."""
    insights = []
    n_rows, n_cols = df.shape
    insights.append(f"O dataset tem {n_rows} linhas e {n_cols} colunas.")

    # Colunas numéricas principais
    numeric = df.select_dtypes(include='number')
    if not numeric.empty:
        # colunas com maior média, maior desvio, mais nulos
        means = numeric.mean().sort_values(ascending=False)
        stds = numeric.std().sort_values(ascending=False)
        nulls = df.isnull().sum().sort_values(ascending=False)

        top_mean_col = means.index[0]
        top_mean_val = means.iloc[0]
        top_std_col = stds.index[0]
        top_std_val = stds.iloc[0]
        top_null_col = nulls.index[0]
        top_null_val = nulls.iloc[0]

        insights.append(f"A coluna com maior média é '{top_mean_col}' (média ≈ {top_mean_val:.2f}).")
        insights.append(f"A coluna com maior variação (desvio padrão) é '{top_std_col}' (σ ≈ {top_std_val:.2f}).")
        if top_null_val > 0:
            insights.append(f"A coluna com mais valores ausentes é '{top_null_col}' (nulos = {top_null_val}).")
        else:
            insights.append("Nenhuma coluna numérica tem valores nulos.")

        # detectar outliers simples (3*std)
        outlier_info = []
        for col in numeric.columns[:5]:  # checar primeiras 5 para não exagerar texto
            col_mean = numeric[col].mean()
            col_std = numeric[col].std()
            if pd.isna(col_std) or col_std == 0:
                continue
            outliers = numeric[(numeric[col] > col_mean + 3*col_std) | (numeric[col] < col_mean - 3*col_std)]
            if not outliers.empty:
                outlier_info.append(f"{col}: {len(outliers)} outliers detectados.")
        if outlier_info:
            insights.append("Outliers detectados (regra 3σ) em: " + "; ".join(outlier_info))

    # Colunas categóricas (top valores)
    cat = df.select_dtypes(include='object')
    if not cat.empty:
        top_freq = []
        for c in cat.columns[:5]:
            top = df[c].value_counts(dropna=True).head(3)
            if not top.empty:
                vals = ", ".join([f"{idx} ({cnt})" for idx, cnt in top.items()])
                top_freq.append(f"{c}: {vals}")
        if top_freq:
            insights.append("Principais categorias (por coluna): " + " | ".join(top_freq))

    # Amarrar tudo com frases curtas
    return "\n".join(insights)


async def call_gemini_placeholder(summary_text: str) -> str:
    """
    Placeholder para integração com Gemini/Google AI.
    Se você configurar GEMINI_KEY e quiser usar API real, substitua esta função
    para chamar a API oficial e retornar o texto de insights.
    Por enquanto retorna um aviso + o resumo local.
    """
    if GEMINI_KEY:
        # Aqui você implementaria a chamada real ao Gemini (ex.: via google generative api).
        # Eu deixo isso como ponto de extensão seguro.
        return ("[Integração Gemini ativada — mas a chamada não foi implementada nesta versão gratuita].\n"
                "Resumo local:\n" + summary_text)
    else:
        return ("[Resumo gerado localmente — sem uso de API externa]\n\n" + summary_text)

# ================================
# FUNÇÕES DE RELATÓRIO E GRAFICOS
# ================================
def make_charts(df: pd.DataFrame):
    numeric_cols = df.select_dtypes(include='number').columns
    chart_files = []
    for col in numeric_cols:
        plt.figure(figsize=(6,4))
        # gerar cor aleatória
        color = (random.random(), random.random(), random.random())
        try:
            df[col].dropna().plot(kind='hist', color=color, title=f'Distribuição de {col}')
            plt.xlabel(col)
            plt.ylabel("Frequência")
            plt.tight_layout()
            filename = f"chart_{col}.png"
            plt.savefig(filename)
            plt.close()
            chart_files.append((filename, col))
        except Exception:
            plt.close()
            continue
    return chart_files

def create_pdf(resumo_text: str, insights_text: str, chart_files, output_filename="relatorio_free.pdf"):
    pdf = FPDF()
    pdf.set_auto_page_break(auto=True, margin=15)
    pdf.add_page()
    pdf.set_font("Arial", 'B', 16)
    pdf.multi_cell(0, 10, "Relatório de Dados", align='C')
    pdf.ln(4)

    # Resumo estatístico
    pdf.set_font("Arial", 'B', 12)
    pdf.multi_cell(0, 8, "Resumo Estatístico")
    pdf.set_font("Arial", '', 10)
    for chunk in textwrap.wrap(resumo_text, 1000):
        pdf.multi_cell(0, 6, chunk)
    pdf.ln(4)

    # Insights
    pdf.set_font("Arial", 'B', 12)
    pdf.multi_cell(0, 8, "Insights Automáticos")
    pdf.set_font("Arial", '', 10)
    for chunk in textwrap.wrap(insights_text, 1000):
        pdf.multi_cell(0, 6, chunk)
    pdf.ln(4)

    # Gráficos
    for filename, col in chart_files:
        pdf.add_page()
        pdf.set_font("Arial", 'B', 12)
        pdf.multi_cell(0, 8, f"Gráfico: Distribuição de {col}")
        try:
            pdf.image(filename, x=10, y=None, w=180)
        except Exception:
            pdf.multi_cell(0, 6, "[Erro ao inserir imagem no PDF]")
    pdf.output(output_filename)
    return output_filename

# ================================
# HANDLERS TELEGRAM
# ================================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Olá! 👋\nEnvie um arquivo CSV ou XLSX para análise.\n"
        "Este bot gera estatísticas, gráficos e um PDF com insights (versão gratuita, sem APIs pagas)."
    )

async def text_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Para gerar relatórios envie um arquivo CSV ou XLSX.\n"
        "Se preferir, salve o arquivo do WhatsApp no seu iPhone e encaminhe aqui no Telegram."
    )

async def handle_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # baixa o arquivo
    doc = update.message.document
    await update.message.reply_text("Recebido! Processando o arquivo, aguarde... ⏳")
    try:
        tfile = await doc.get_file()
        data = await tfile.download_as_bytearray()
    except Exception as e:
        await update.message.reply_text(f"Erro ao baixar o arquivo: {e}")
        return

    # tentar ler CSV ou Excel
    try:
        if doc.file_name.lower().endswith(".csv"):
            df = pd.read_csv(io.BytesIO(data))
        else:
            df = pd.read_excel(io.BytesIO(data))
    except Exception as e:
        await update.message.reply_text(f"Não consegui ler o arquivo. Verifique se é CSV ou XLSX. Erro: {e}")
        return

    # gerar resumo estatístico (texto)
    try:
        resumo = df.describe(include='all').to_string()
    except Exception:
        resumo = f"Resumo não pôde ser gerado automaticamente para este arquivo. Forma: {type(df)}"

    # gerar insights locais
    local_insights = generate_local_insights(df)

    # se houver GEMINI_KEY, chamar stub/placeholder (pode ser substituído por integração real)
    insights_full = await call_gemini_placeholder(local_insights)

    # enviar resumo e insights no chat (limitar tamanho de mensagem)
    await update.message.reply_text("📋 Resumo estatístico (primeira parte):")
    await update.message.reply_text(resumo[:3000])
    await update.message.reply_text("🧠 Insights automáticos:")
    await update.message.reply_text(insights_full[:3000])

    # gerar gráficos
    charts = make_charts(df)
    for filename, col in charts:
        try:
            await update.message.reply_photo(photo=open(filename, "rb"))
        except Exception:
            pass

    # gerar PDF
    pdf_file = create_pdf(resumo, insights_full, charts, output_filename="relatorio_free.pdf")

    await update.message.reply_text("✅ PDF pronto — abaixo está o arquivo. Toque e compartilhe via WhatsApp se desejar.")
    try:
        await update.message.reply_document(document=open(pdf_file, "rb"))
    except Exception as e:
        await update.message.reply_text(f"Erro ao enviar o PDF: {e}")

# ================================
# START BOT
# ================================
def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.Document.ALL, handle_file))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_message))
    print("Bot iniciado.")
    app.run_polling()

if __name__ == "__main__":
    main()
