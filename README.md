# OdontoPro - Documentação do Projeto

Bem-vindo ao **OdontoPro**! Este documento foi preparado para guiar você, desenvolvedor(a), desde a configuração inicial até o entendimento profundo da arquitetura e do código.

---

## 1. Passo a Passo para Rodar o Projeto

Siga estas instruções para colocar a aplicação em execução no seu ambiente local.

### Pré-requisitos

- **Node.js**: Certifique-se de ter o Node.js instalado (versão 18 ou superior recomendada).
- **Gerenciador de Pacotes**: npm, yarn ou pnpm.
- **Banco de Dados**: PostgreSQL (necessário para o Prisma).

### Instalação e Execução

1.  **Instalar Dependências:**
    Baixe as bibliotecas listadas no `package.json`.

    ```bash
    npm install
    # ou
    yarn install
    ```

2.  **Configurar Variáveis de Ambiente:**
    Crie um arquivo `.env` na raiz do projeto. Você precisará configurar a conexão com o banco de dados e segredos de autenticação.
    Exemplo:

    ```env
    DATABASE_URL="postgresql://user:password@localhost:5432/odontopro"
    AUTH_SECRET="seu-segredo-super-secreto"
    # Chaves do Stripe (opcional para rodar, necessário para pagamentos)
    STRIPE_API_KEY="..."
    ```

3.  **Configurar o Banco de Dados:**
    O projeto utiliza o Prisma ORM. Execute o comando abaixo para criar as tabelas no seu banco de dados baseadas no schema definido em `prisma/schema.prisma`:

    ```bash
    npm run prisma:migrate
    ```

4.  **Rodar o Servidor de Desenvolvimento:**
    Inicie a aplicação localmente.

    ```bash
    npm run dev
    ```

5.  **Acessar:**
    Abra o navegador em http://localhost:3000.

---

## 2. Estrutura e Arquitetura do Projeto

O projeto utiliza **Next.js** com **App Router**, uma arquitetura moderna que unifica Frontend e Backend no mesmo repositório (Monolito Modular).

### Tecnologias Principais

- **Framework**: Next.js 16 (React Server Components, API Routes).
- **Linguagem**: TypeScript (Tipagem estática).
- **Banco de Dados**: Prisma ORM (Camada de abstração de dados).
- **Autenticação**: NextAuth.js (Auth.js v5) com adaptador Prisma.
- **Estilização**: Tailwind CSS (Utility-first CSS).
- **Pagamentos**: Stripe (Integração via SDK e Webhooks).
- **Validação**: Zod (Validação de schemas).

### Mapa de Pastas

```
src/
├── app/                   # O coração da aplicação (Rotas e Páginas)
│   ├── api/               # Backend (API Routes para chamadas HTTP)
│   │   └── clinic/        # Funcionalidades específicas da clínica
│   └── (public)/          # Rotas públicas
├── components/            # Componentes visuais reutilizáveis (Botões, Inputs, Labels)
│   └── ui/                # Componentes de interface genéricos
├── lib/                   # Configurações de bibliotecas (Prisma, Auth)
```

---

## 3. Explicação Detalhada dos Arquivos e Rotas

Abaixo, uma análise técnica profunda dos arquivos principais identificados no contexto do projeto.

### 📂 Backend: `src/app/api/clinic/appointments/route.ts`

**Tipo:** Rota de API (Next.js Route Handler).
**Caminho da Rota:** `/api/clinic/appointments`
**Método HTTP:** `GET`

**Responsabilidade:**
Este arquivo é o endpoint responsável por fornecer os dados de agendamentos de uma clínica para o frontend. Ele filtra os agendamentos com base em uma data específica fornecida e garante que apenas a clínica autenticada acesse seus próprios dados.

**Detalhamento do Código:**

1.  **Autenticação (`auth`)**:
    - A função `GET` é envolvida pelo wrapper `auth`. Isso injeta a sessão do usuário na requisição.
    - `if (!request.auth)`: Verifica explicitamente se há uma sessão válida. Se não houver, retorna `401 Unauthorized`. Isso impede acesso não autorizado.
2.  **Extração de Dados**:
    - `searchParams.get("date")`: A data é obtida dos parâmetros da URL (Query Param), pois é um filtro dinâmico (ex: `?date=2024-02-20`).
    - `request.auth.user.id`: O ID da clínica é obtido **da sessão segura** (token criptografado), e **não** da URL. Isso é crucial para segurança, impedindo que um usuário acesse dados de outra clínica manipulando IDs.
3.  **Validação**:
    - Verifica se a data e o ID da clínica estão presentes. Retorna `400 Bad Request` se faltar algo.
4.  **Lógica de Data**:
    - O banco de dados armazena datas com horário (Timestamp). Para buscar "todos os agendamentos do dia X", o código cria um intervalo de tempo:
      - `startDate`: Início do dia (00:00:00.000).
      - `endDate`: Fim do dia (23:59:59.999).
    - Isso permite usar os operadores `gte` (maior ou igual) e `lte` (menor ou igual) do Prisma para encontrar registros dentro desse dia.
5.  **Consulta ao Banco (Prisma)**:
    - `prisma.appointment.findMany(...)`: Executa a busca na tabela `Appointment`.
    - `where`: Aplica os filtros de `userId` (clínica) e o intervalo de data calculado.
    - `include: { service: true }`: Realiza um _Eager Loading_ (similar a um JOIN no SQL) para trazer os detalhes do serviço (nome, preço, duração) atrelado a cada agendamento na mesma consulta.

### 📂 Banco de Dados: `prisma/schema.prisma`

**Tipo:** Definição de Schema do Prisma.
**Responsabilidade:** Definir a estrutura do banco de dados, tabelas (models) e relacionamentos.

**Principais Models:**

- **User**: Representa a clínica ou profissional. Possui relacionamentos com `Subscription`, `Service`, `Appointment`, etc.
- **Appointment**: Representa um agendamento.
  - Relaciona-se com `Service` (qual serviço será feito).
  - Relaciona-se com `User` (a qual clínica pertence).
  - Campos: `appointmentDate` (data/hora), `name`, `email` (dados do paciente).
- **Service**: Serviços oferecidos pela clínica (ex: "Limpeza", "Canal"). Tem preço e duração.
- **Subscription**: Gerencia a assinatura da clínica (Plano Basic ou Professional).

### 📂 Componente UI: `src/components/ui/label-subscription.tsx`

**Tipo:** Componente React (Interface).
**Responsabilidade:** Exibir um banner de alerta visual para o usuário quando há problemas com sua assinatura.

**Detalhamento do Código:**

1.  **Props**: Recebe `expired: boolean`. Isso torna o componente "burro" (stateless), ele apenas exibe o que recebe; a lógica de verificar se expirou fica no componente pai ou no backend.
2.  **Renderização Condicional**:
    - Usa um operador ternário (`expired ? ... : ...`) para alternar a mensagem.
    - **Cenário 1 (Expired)**: "Seu plano expirou...".
    - **Cenário 2 (Limite)**: "Você excedeu o limite...".
3.  **UX (Experiência do Usuário)**:
    - Usa cores de alerta (`bg-red-400`) para chamar atenção imediata.
    - Inclui um botão de ação (`Link` para `/dashboard/plans`) para que o usuário possa resolver o problema imediatamente (Upsell/Renovação).

### 📂 Configuração: `package.json`

**Tipo:** Manifesto do Projeto Node.js.
**Responsabilidade:** Definir dependências, scripts de execução e metadados.

**Destaques Técnicos:**

- **`scripts`**:
  - `dev`: Inicia o servidor Next.js com suporte a hot-reload.
  - `prisma:migrate`: Comando essencial para aplicar mudanças no schema do banco de dados.
  - `stripe:listen`: Ferramenta de desenvolvimento para receber eventos de pagamento do Stripe na sua máquina local (Webhooks).
- **`dependencies`**:
  - `@auth/prisma-adapter`: Conecta o NextAuth ao banco de dados via Prisma.
  - `zod` + `react-hook-form`: Combo padrão da indústria para criar formulários robustos com validação.
  - `@tanstack/react-query`: Gerenciamento de estado assíncrono (cache, refetching) para dados vindos da API.
