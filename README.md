# KR4US3 Chat Gabarito

Conjunto de instruções criado para orientar o comportamento de assistentes de IA durante conversas, com foco em qualidade da decisão, independência intelectual, precisão, eficiência, continuidade e redução de respostas superficiais ou excessivamente complacentes.

O conjunto é composto por duas camadas complementares:

- `KR4US3-InstrucoesPersonalizadas.md`
- `KR4US3-GabaritoChat.md`

## Objetivo

O objetivo não é apenas alterar o estilo de escrita da IA.

As instruções procuram influenciar a forma como ela trabalha, incluindo:

- foco no resultado real do usuário;
- resistência à concordância automática;
- proporcionalidade entre complexidade do problema e profundidade da resposta;
- diferenciação entre fatos, hipóteses e recomendações;
- verificação factual quando necessária;
- uso adequado de ferramentas;
- análise de consequências de segunda ordem;
- diagnóstico antes de alteração;
- preferência por ações reversíveis;
- consideração de custo total e operação futura;
- continuidade entre etapas de um trabalho;
- recomendação explícita quando houver base suficiente;
- redução de perguntas e trabalho desnecessários para o usuário.

---

# Arquivos

## 1. KR4US3-InstrucoesPersonalizadas.md

Este arquivo contém a camada global e compacta das instruções.

Ele foi projetado para ser colocado nas instruções personalizadas do ChatGPT e permanecer ativo durante as conversas.

### Como usar

Copie o conteúdo do arquivo e adicione em:

`Configurações > Personalização > Instruções personalizadas`

Essa camada deve permanecer configurada continuamente.

Ela contém os princípios que fazem sentido independentemente do assunto da conversa.

---

## 2. KR4US3-GabaritoChat.md

Este é o gabarito completo.

Ele contém regras mais detalhadas de raciocínio e execução, incluindo protocolos para:

- decisões;
- diagnóstico;
- troubleshooting;
- planejamento;
- pesquisa;
- arquitetura;
- código;
- documentação;
- avaliação de riscos;
- reversibilidade;
- pré-mortem;
- verificação final.

### Como usar

Anexe o arquivo `KR4US3-GabaritoChat.md` no início de uma nova conversa.

Em seguida, envie uma instrução simples como:

> Leia e aplique o gabarito anexado durante toda esta conversa.

O próprio arquivo instrui a IA a confirmar sua ativação apenas na primeira resposta com algo semelhante a:

> Gabarito em uso.

Não é necessário repetir essa instrução nas mensagens seguintes.

---

# Uso recomendado

A configuração recomendada é:

```text
INSTRUÇÕES PERSONALIZADAS
        │
        │ sempre ativas
        ▼
KR4US3-InstrucoesPersonalizadas.md
        │
        │
        ▼
      CHAT
        │
        │ quando maior rigor for desejado
        ▼
KR4US3-GabaritoChat.md
