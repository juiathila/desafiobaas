# Relatório de Correções — Nexus dos Heróis

## BUG #01 — Login Silencia Erros
**Arquivo:** `src/app/(auth)/login/page.tsx`

- **Acontecia:** login errado não mostrava mensagem nenhuma; botão travava em "Entrando...".
- **Causa:** `catch` vazio, o erro do Firebase era descartado.
- **Correção:**
  ```ts
  // antes
  } catch {
  }

  // depois
  } catch (err) {
    const msg = err instanceof Error ? err.message : "Erro desconhecido";
    if (msg.includes("invalid-credential") || msg.includes("wrong-password")) {
      setErro("E-mail ou senha incorretos.");
    } else if (msg.includes("user-not-found")) {
      setErro("Nenhuma conta encontrada com este e-mail.");
    } else {
      setErro("Erro ao entrar. Tente novamente.");
    }
  }
  ```

## BUG #02 — Middleware com Condição Invertida
**Arquivo:** `middleware.ts`

- **Acontecia:** quem não estava logado acessava rotas protegidas; quem estava logado era barrado.
- **Causa:** condição `if (token)` redirecionava justamente quem tinha sessão.
- **Correção:** `if (token)` → `if (!token)`.

## BUG #03 — Confirmação de Senha Compara com Campo Errado
**Arquivo:** `src/app/(auth)/cadastro/page.tsx`

- **Acontecia:** cadastro aceitava senha e confirmação diferentes.
- **Causa:** comparação usava a variável `nome` no lugar de `confirmarSenha`.
- **Correção:** `if (senha !== nome)` → `if (senha !== confirmarSenha)`.

## BUG #04 — Listagem sem Filtro por Usuário
**Arquivo:** `src/services/personagens.ts`

- **Acontecia:** dashboard mostrava personagens de todos os usuários.
- **Causa:** query sem cláusula `where` pelo `userId`.
- **Correção:**
  ```ts
  // antes
  const q = query(collection(db, "personagens"));

  // depois
  const q = query(collection(db, "personagens"), where("userId", "==", uid));
  ```

## BUG #05 — Criação na Coleção Errada
**Arquivo:** `src/services/personagens.ts`

- **Acontecia:** personagem criado não aparecia no dashboard.
- **Causa:** `addDoc` gravava em `"personagem"` (singular); a listagem lê de `"personagens"` (plural).
- **Correção:** `collection(db, "personagem")` → `collection(db, "personagens")`.

## BUG #06 — Equipar Item Apaga o Documento Inteiro
**Arquivo:** `src/services/personagens.ts`

- **Acontecia:** equipar um item apagava os outros equipamentos e campos do personagem.
- **Causa:** `setDoc` substitui o documento inteiro; recebia só o campo do slot.
- **Correção:** `setDoc(...)` → `updateDoc(...)` (e removido o import não usado de `setDoc`).

## BUG #07 — Exclusão pelo Índice da Lista
**Arquivo:** `src/services/personagens.ts` e `src/app/dashboard/page.tsx`

- **Acontecia:** deletar um personagem removia o errado ou dava erro de documento inexistente.
- **Causa:** função usava o índice da lista (`0, 1, 2...`) como ID do documento, em vez do ID real do Firestore.
- **Correção:**
  ```ts
  // antes
  export async function deletarPersonagem(personagem: Personagem, indice: number) {
    await deleteDoc(doc(db, "personagens", String(indice)));
  }

  // depois
  export async function deletarPersonagem(personagem: Personagem) {
    await deleteDoc(doc(db, "personagens", personagem.id));
  }
  ```
  Chamada no dashboard ajustada de `deletarPersonagem(personagem, indice)` para `deletarPersonagem(personagem)`.

## BUG #08 — Regras de Segurança Totalmente Abertas
**Arquivo:** `firestore.rules`

- **Acontecia:** qualquer pessoa, sem login, podia ler, criar, editar ou excluir qualquer documento.
- **Causa:** regra `allow read, write: if true`.
- **Correção:**
  ```
  match /personagens/{personagemId} {
    allow read: if request.auth != null &&
                request.auth.uid == resource.data.userId;
    allow create: if request.auth != null &&
                  request.auth.uid == request.resource.data.userId;
    allow update, delete: if request.auth != null &&
                          request.auth.uid == resource.data.userId;
  }
  ```

---

## Validação

- `npm run build`: TypeScript compila sem erros. A etapa de pré-renderização só falha por falta de credenciais reais no `.env.local` (Firebase ainda não configurado no ambiente de teste).
- Testes manuais end-to-end (login, cadastro, dashboard, criar/equipar/deletar personagem) e publicação das `firestore.rules` no console do Firebase: pendentes até o `.env.local` estar configurado com um projeto Firebase real.

## Screenshots

_A adicionar após o `.env.local` estar configurado: print de cada bug antes (com o app rodando na versão anterior) e depois (corrigido)._
