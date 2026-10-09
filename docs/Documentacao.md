# Documentação — Utilização de GitHub Secrets

## 1. Problema encontrado

No arquivo `.github/workflows/main.yml` (chamado `pipeline.yml` no enunciado), a variável `MENSAGEM_SECRETA` estava escrita diretamente no código do workflow:

```yaml
env:
  MENSAGEM_SECRETA: "Minha mensagem secreta"
```

Como o workflow é versionado no repositório, o valor sensível ficava exposto junto com o código.

### Respostas da seção 3

1. **Onde o segredo está armazenado?** Em texto puro, dentro do arquivo `main.yml`, que faz parte do repositório Git (e de todo o seu histórico de commits).
2. **Quem poderia visualizar?** Qualquer pessoa com acesso de leitura ao repositório. Se ele for público, qualquer pessoa na internet. Também quem clonar ou fizer fork do projeto.
3. **Por que não é recomendado?** Porque mistura configuração sensível com código. O valor fica no histórico do Git mesmo que seja removido depois, não pode ser trocado sem novo commit e não tem controle de acesso próprio.
4. **E se fosse uma senha ou token real?** Poderia ser usada por terceiros para acessar bancos de dados, APIs ou servidores, causando vazamento de dados, uso indevido de recursos e custos. A credencial teria de ser revogada e trocada, pois continuaria no histórico.

## 2. Secret utilizado

- **Nome:** `MENSAGEM_SECRETA`
- **Valor de exemplo:** `Esta mensagem está protegida pelo GitHub Secrets` (o valor real fica apenas no GitHub)
- **Onde foi criado:** Settings → Security → Secrets and variables → Actions → aba Secrets → New repository secret

## 3. Alteração na pipeline

O valor fixo foi substituído pela referência ao Secret:

```yaml
- name: Executar aplicação
  env:
    MENSAGEM_SECRETA: ${{ secrets.MENSAGEM_SECRETA }}
  run: |
    cd app
    node server.js
```

## 4. Como o Secret chegou até a aplicação

**Secret → GitHub → GitHub Actions → Variável de ambiente → Aplicação**

1. O Secret `MENSAGEM_SECRETA` é cadastrado nas configurações do repositório e armazenado de forma criptografada pelo GitHub.
2. Quando o push na `main` dispara o workflow, o GitHub Actions resolve a expressão `${{ secrets.MENSAGEM_SECRETA }}` e obtém o valor.
3. Esse valor é injetado como variável de ambiente `MENSAGEM_SECRETA` no passo "Executar aplicação".
4. O `server.js` lê o valor com `process.env.MENSAGEM_SECRETA`, sem nunca conhecer o segredo no código.

O GitHub também mascara o valor do Secret nos logs, exibindo `***` caso ele apareça. Além disso, a aplicação imprime apenas a quantidade de caracteres, nunca o conteúdo.

## 5. Evidência de execução

Teste local simulando a injeção feita pelo Actions:

```text
$ MENSAGEM_SECRETA="Esta mensagem está protegida pelo GitHub Secrets" node server.js
Secret recebido pela aplicação com sucesso.
Quantidade de caracteres: 48
```

Sem a variável definida, a aplicação falha como esperado:

```text
$ node server.js
MENSAGEM_SECRETA não foi configurada.
(exit code 1)
```

**Execução no GitHub Actions:** ![alt text](<Captura de tela 2026-10-09 201737.png>)

### Checklist de validação

- [x] A aplicação continua recebendo a mensagem (`Secret recebido pela aplicação com sucesso.`)
- [x] O `main.yml` não contém mais o valor real do segredo
- [x] O valor do Secret não está no código
- [x] O valor sensível não é exibido nos logs (apenas o número de caracteres)

## 6. Respostas das questões (seção 7)

1. **Diferença entre valor no `pipeline.yml` e no GitHub Secrets?** No arquivo, o valor fica visível a quem acessa o repositório e no histórico do Git. No GitHub Secrets, ele é armazenado criptografado fora do código, não pode ser lido de volta pela interface e é mascarado nos logs.
2. **Onde o valor do Secret fica configurado?** Nas configurações do repositório: Settings → Secrets and variables → Actions → Secrets.
3. **O valor do Secret deve ser colocado no código da aplicação?** Não. O código só deve ler o valor via variável de ambiente, sem conhecê-lo.
4. **Qual a função de `process.env.MENSAGEM_SECRETA`?** Ler, dentro do Node.js, a variável de ambiente `MENSAGEM_SECRETA` definida pelo ambiente de execução (aqui, o GitHub Actions). É o ponto em que o segredo chega à aplicação.
5. **Por que usar Secrets é importante em CI/CD?** Pipelines usam credenciais (tokens, senhas, chaves de deploy). Secrets evitam o vazamento no repositório, permitem trocar valores sem alterar código, separam ambientes e reduzem o impacto de acessos indevidos ao código.
6. **Por que é inadequado colocar a senha de um banco de dados no `.yml`?** Porque qualquer pessoa com acesso ao repositório (ou ao histórico) poderia ler a senha e acessar o banco. Além disso, a senha ficaria presa ao código, dificultando a rotação, e um vazamento exigiria trocar a credencial e limpar o histórico.