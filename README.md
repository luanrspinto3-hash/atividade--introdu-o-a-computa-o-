# atividade--introdu-o-a-computa-o-

import json
import matplotlib.pyplot as plt

from rich.console import Console
from rich.panel import Panel
from rich.table import Table
from rich.text import Text
from rich import box
from rich.prompt import Prompt, IntPrompt
from rich.align import Align
from rich.progress import Progress, BarColumn, TextColumn

console = Console()

ARQUIVO = "financeiro.json"

dados_financeiros = {
    "saldo": 0.0,
    "orcamentos": {
        "Alimentação": 500.0,
        "Transporte": 200.0,
        "Lazer": 150.0,
        "Contas": 1000.0
    },
    "movimentacoes": []
}


def carregar_dados():
    try:
        with open(ARQUIVO, "r", encoding="utf-8") as arquivo:
            return json.load(arquivo)
    except FileNotFoundError:
        return dados_financeiros.copy()


def salvar_dados(dados):
    with open(ARQUIVO, "w", encoding="utf-8") as arquivo:
        json.dump(dados, arquivo, indent=4, ensure_ascii=False)


def moeda(valor):
    return f"R$ {valor:,.2f}".replace(",", "X").replace(".", ",").replace("X", ".")


def cabecalho():
    console.clear()

    titulo = Text()
    titulo.append("💰 GERENCIADOR FINANCEIRO\n", style="bold bright_cyan")
    titulo.append("Controle inteligente das suas finanças", style="italic white")

    console.print(
        Panel(
            Align.center(titulo),
            border_style="bright_cyan",
            box=box.DOUBLE,
            padding=(1, 2)
        )
    )


def painel_resumo(dados):
    receitas = sum(
        m["valor"] for m in dados["movimentacoes"]
        if m["tipo"] == "Receita"
    )

    despesas = sum(
        m["valor"] for m in dados["movimentacoes"]
        if m["tipo"] == "Despesa"
    )

    tabela = Table(
        box=box.ROUNDED,
        expand=True,
        border_style="blue"
    )

    tabela.add_column("Indicador", style="bold white")
    tabela.add_column("Valor", justify="right")

    tabela.add_row("💰 Saldo atual", f"[bold cyan]{moeda(dados['saldo'])}[/]")
    tabela.add_row("📈 Total de receitas", f"[green]{moeda(receitas)}[/]")
    tabela.add_row("📉 Total de despesas", f"[red]{moeda(despesas)}[/]")

    console.print(tabela)


def adicionar_movimentacao(dados):
    cabecalho()

    console.print(Panel("[bold cyan]NOVA MOVIMENTAÇÃO[/]", border_style="cyan"))

    tipo = Prompt.ask(
        "Tipo",
        choices=["Receita", "Despesa"],
        default="Despesa"
    )

    valor = Prompt.ask("Valor (R$)").replace(",", ".")

    try:
        valor = float(valor)
        if valor <= 0:
            raise ValueError
    except ValueError:
        console.print("[bold red]Valor inválido![/]")
        Prompt.ask("Pressione ENTER para continuar")
        return

    categorias = list(dados["orcamentos"].keys())

    categoria = Prompt.ask(
        "Categoria",
        choices=categorias
    )

    data = Prompt.ask("Data (DD/MM/AAAA)")
    descricao = Prompt.ask("Descrição")

    movimentacao = {
        "tipo": tipo,
        "valor": valor,
        "categoria": categoria,
        "data": data,
        "descricao": descricao
    }

    dados["movimentacoes"].append(movimentacao)

    if tipo == "Receita":
        dados["saldo"] += valor
    else:
        dados["saldo"] -= valor

    salvar_dados(dados)

    console.print(
        Panel(
            "[bold green]Movimentação cadastrada com sucesso![/]",
            border_style="green"
        )
    )

    verificar_alerta(dados, categoria)
    Prompt.ask("Pressione ENTER para continuar")


def verificar_alerta(dados, categoria):
    limite = dados["orcamentos"].get(categoria, 0)

    gasto = sum(
        m["valor"] for m in dados["movimentacoes"]
        if m["categoria"] == categoria
        and m["tipo"] == "Despesa"
    )

    if limite > 0 and gasto > limite:
        console.print(
            Panel(
                f"[bold red]ATENÇÃO! Orçamento ultrapassado![/]\n"
                f"Categoria: {categoria}\n"
                f"Limite: {moeda(limite)}\n"
                f"Gasto: {moeda(gasto)}",
                border_style="red"
            )
        )


def listar_movimentacoes(dados):
    cabecalho()

    tabela = Table(
        title="📋 HISTÓRICO DE MOVIMENTAÇÕES",
        box=box.ROUNDED,
        expand=True,
        border_style="cyan"
    )

    tabela.add_column("Nº", justify="center")
    tabela.add_column("Data")
    tabela.add_column("Tipo")
    tabela.add_column("Descrição")
    tabela.add_column("Categoria")
    tabela.add_column("Valor", justify="right")

    for i, m in enumerate(dados["movimentacoes"], 1):
        cor = "green" if m["tipo"] == "Receita" else "red"
        sinal = "+" if m["tipo"] == "Receita" else "-"

        tabela.add_row(
            str(i),
            m["data"],
            f"[{cor}]{m['tipo']}[/]",
            m["descricao"],
            m["categoria"],
            f"[{cor}]{sinal}{moeda(m['valor'])}[/]"
        )

    console.print(tabela)

    Prompt.ask("Pressione ENTER para continuar")


def remover_movimentacao(dados):
    listar_movimentacoes(dados)

    movs = dados["movimentacoes"]

    if not movs:
        return

    try:
        numero = IntPrompt.ask("Número da movimentação para remover")

        if numero < 1 or numero > len(movs):
            raise ValueError

        removida = movs.pop(numero - 1)

        if removida["tipo"] == "Receita":
            dados["saldo"] -= removida["valor"]
        else:
            dados["saldo"] += removida["valor"]

        salvar_dados(dados)

        console.print("[bold green]Movimentação removida com sucesso![/]")

    except ValueError:
        console.print("[bold red]Número inválido![/]")

    Prompt.ask("Pressione ENTER para continuar")


def relatorio_orcamento(dados):
    cabecalho()

    console.print(
        Panel("[bold cyan]RELATÓRIO DE ORÇAMENTOS[/]", border_style="cyan")
    )

    tabela = Table(box=box.ROUNDED, expand=True)

    tabela.add_column("Categoria")
    tabela.add_column("Limite", justify="right")
    tabela.add_column("Gasto", justify="right")
    tabela.add_column("Utilização", justify="right")
    tabela.add_column("Status")

    for categoria, limite in dados["orcamentos"].items():

        gasto = sum(
            m["valor"] for m in dados["movimentacoes"]
            if m["categoria"] == categoria
            and m["tipo"] == "Despesa"
        )

        percentual = gasto / limite * 100 if limite > 0 else 0

        if gasto > limite:
            status = "[bold red]ESTOURADO[/]"
        elif gasto == limite:
            status = "[yellow]NO LIMITE[/]"
        else:
            status = "[green]DENTRO DO LIMITE[/]"

        tabela.add_row(
            categoria,
            moeda(limite),
            moeda(gasto),
            f"{percentual:.1f}%",
            status
        )

    console.print(tabela)

    Prompt.ask("Pressione ENTER para continuar")


def gerar_graficos(dados):
    despesas = [
        m for m in dados["movimentacoes"]
        if m["tipo"] == "Despesa"
    ]

    if not despesas:
        console.print("[red]Nenhuma despesa cadastrada.[/]")
        Prompt.ask("Pressione ENTER para continuar")
        return

    periodo = Prompt.ask(
        "Filtrar por mês/ano (MM/AAAA) ou ENTER para todos",
        default=""
    )

    if periodo:
        despesas = [
            d for d in despesas
            if len(d["data"].split("/")) == 3
            and "/".join(d["data"].split("/")[1:]) == periodo
        ]

    if not despesas:
        console.print("[red]Nenhuma despesa encontrada.[/]")
        Prompt.ask("Pressione ENTER para continuar")
        return

    gastos = {}

    for d in despesas:
        categoria = d["categoria"]
        gastos[categoria] = gastos.get(categoria, 0) + d["valor"]

    categorias = list(gastos.keys())
    valores = list(gastos.values())

    limites = [
        dados["orcamentos"].get(c, 0)
        for c in categorias
    ]

    plt.style.use("dark_background")

    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(13, 5))

    fig.patch.set_facecolor("#151b2b")

    ax1.pie(
        valores,
        labels=categorias,
        autopct="%1.1f%%",
        startangle=140
    )

    ax1.set_title("Distribuição das Despesas")

    indices = range(len(categorias))
    largura = 0.35

    ax2.bar(
        [i - largura/2 for i in indices],
        valores,
        largura,
        label="Gasto Real",
        color="#e74c3c"
    )

    ax2.bar(
        [i + largura/2 for i in indices],
        limites,
        largura,
        label="Orçamento",
        color="#3498db"
    )

    ax2.set_xticks(list(indices))
    ax2.set_xticklabels(categorias)
    ax2.set_title("Gastos x Orçamento")
    ax2.legend()

    plt.tight_layout()
    plt.show()


def menu_principal():
    dados = carregar_dados()

    while True:
        cabecalho()

        painel_resumo(dados)

        console.print()

        menu = Table(
            title="🏠 MENU PRINCIPAL",
            box=box.ROUNDED,
            border_style="bright_blue",
            expand=True
        )

        menu.add_column("Opção", justify="center", style="bold cyan")
        menu.add_column("Funcionalidade")

        menu.add_row("1", "➕ Adicionar movimentação")
        menu.add_row("2", "📋 Listar movimentações")
        menu.add_row("3", "🗑️ Remover movimentação")
        menu.add_row("4", "📊 Relatório de orçamentos")
        menu.add_row("5", "📈 Visualizar gráficos")
        menu.add_row("6", "🚪 Sair")

        console.print(menu)

        opcao = Prompt.ask(
            "\n[bold cyan]Escolha uma opção[/]",
            choices=["1", "2", "3", "4", "5", "6"]
        )

        if opcao == "1":
            adicionar_movimentacao(dados)

        elif opcao == "2":
            listar_movimentacoes(dados)

        elif opcao == "3":
            remover_movimentacao(dados)

        elif opcao == "4":
            relatorio_orcamento(dados)

        elif opcao == "5":
            gerar_graficos(dados)

        elif opcao == "6":
            console.print(
                Panel(
                    "[bold green]Sistema encerrado. Até logo![/]",
                    border_style="green"
                )
            )
            break


if __name__ == "__main__":
    menu_principal()