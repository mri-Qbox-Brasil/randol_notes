# randol_notes — Manual

Bloco de notas como item do ox_inventory: cada bloco tem um ID próprio, guarda várias notas no banco e permite destacar uma folha, que vira um item entregável a outro jogador.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Itens do ox_inventory](#itens-do-ox_inventory)
4. [Fluxo de uso](#fluxo-de-uso)
5. [Banco de dados](#banco-de-dados)
6. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
7. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core`, `es_extended` **ou** `ND_Core` | Sim | Bridge automático em `bridge/`. O bridge ND exige ND_Core 2.0.0+ |
| `ox_lib` | Sim | `lib.callback`, `lib.registerContext`, `lib.inputDialog` |
| `ox_inventory` | Sim | Obrigatório de verdade: `sv_notes.lua` usa `exports.ox_inventory` direto e registra o hook `createItem`. Não funciona com outro inventário |
| `oxmysql` | Sim | Persistência das notas na tabela `rnotes` |

---

## Instalação

1. Copie a pasta `randol_notes` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure randol_notes
   ```
3. Cadastre os itens `notepad` e `tornnote` no `ox_inventory` (ver abaixo) e copie `images/notepad.png` e `images/tornnote.png` para a pasta de imagens do inventário.
4. Não é preciso importar SQL: a tabela `rnotes` é criada automaticamente no start do recurso.

---

## Itens do ox_inventory

Em `ox_inventory/data/items.lua`:

```lua
["notepad"] = {
    label = "Notepad",
    weight = 0,
    stack = false,
    close = true,
    consume = 0,
    description = "Sometimes handy to remember something :)",
    server = {
        export = 'randol_notes.notepad',
    },
},

["tornnote"] = {
    label = "Torn Note",
    weight = 0,
    stack = false,
    close = false,
},
```

Se renomear o item `notepad`, é preciso trocar o nome também no `itemFilter` do hook `createItem`, no fim de `sv_notes.lua`:

```lua
}, {
    print = false,
    itemFilter = {
        notepad = true
    }
})
```

---

## Fluxo de uso

1. **Criação do bloco** — o hook `createItem` do ox_inventory intercepta cada `notepad` criado, gera um `noteid` alfanumérico único de 8 caracteres, marca o item como `unique`, escreve a descrição (`Barcode: <noteid>` + dono) e insere a linha correspondente em `rnotes`.
2. **Usar o item** abre o menu "Bloco de Notas" com a animação de prancheta (props `prop_notepad_01` e `prop_pencil_01` na mão).
3. **Nova nota** — abre um input com o texto, a opção **assinar** e a opção **destacar**:
   - sem destacar: a nota é gravada no bloco (JSON na coluna `notes`);
   - destacando: nada é gravado — o jogador recebe direto um item `tornnote` com o texto na metadata.
4. **Minhas anotações** — lista as notas do bloco (as mais recentes primeiro), com preview de 10 caracteres, data e se está assinada ou anônima. O texto completo aparece na metadata do contexto.
5. Ao abrir uma nota salva, é possível **excluir** ou **destacar** — destacar remove a nota do bloco e devolve um item `tornnote` com data, autor (nome do personagem se assinada, senão `Unsigned`) e o texto na descrição.

O `tornnote` é um item comum: pode ser entregue a outro jogador, e o texto fica visível na descrição do item.

---

## Banco de dados

Criada automaticamente no start do recurso:

```sql
CREATE TABLE IF NOT EXISTS `rnotes` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `noteid` varchar(50) NOT NULL DEFAULT '0',
  `citizenid` varchar(100) NOT NULL DEFAULT '0',
  `notes` longtext NOT NULL,
  PRIMARY KEY (`id`)
);
```

| Coluna | Descrição |
|---|---|
| `noteid` | ID do bloco, igual ao `metadata.noteid` do item. É a chave usada em todas as consultas |
| `citizenid` | Identificador do personagem que criou o bloco |
| `notes` | Array JSON de notas (`id`, `text`, `signed`, `date`) |

As notas ficam vinculadas ao **bloco**, não ao personagem: quem estiver com o item lê o conteúdo.

---

## Entrypoints para outros recursos

O recurso registra o export `notepad`, consumido pelo `ox_inventory` como handler de uso do item:

```lua
exports('notepad', function(event, item, inventory, slot, data)
    -- abre o bloco de notas para inventory.id usando o metadata.noteid
end)
```

---

## Estrutura de arquivos

```
randol_notes/
├── bridge/
│   ├── client/
│   │   ├── esx.lua        — notificação no ESX
│   │   ├── nd.lua         — notificação no ND_Core
│   │   └── qb.lua         — notificação no QB
│   └── server/
│       ├── esx.lua        — GetPlayer, identificador e nome do personagem no ESX
│       ├── nd.lua         — GetPlayer, identificador e nome do personagem no ND_Core
│       └── qb.lua         — GetPlayer, citizenid e nome do personagem no QB
├── cl_notes.lua           — menus de contexto, input de nota, animação e props
├── sv_notes.lua           — hook createItem, CRUD das notas, tornnote, tabela rnotes
├── images/
│   ├── notepad.png        — ícone do item notepad
│   └── tornnote.png       — ícone do item tornnote
└── fxmanifest.lua
```
