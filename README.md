# Base de Conhecimento — Formulários Conectados (JetFormBuilder)

> Objetivo da funcionalidade: **o Formulário [1] é enviado/alterado → o Formulário [2] se atualiza sem reload da página; e o inverso também.** Modelo mental declarado pelo autor: o checkout do WooCommerce (aplicar cupom → o resumo do pedido se atualiza sozinho).
>
> **Fase atual: conhecimento e evidência. Nenhum arquivo criado nos repositórios, nenhuma branch criada.**

---

## 0. Protocolo aplicado

Este documento segue o `core-plugins/README.md`: nada aqui foi assumido por conhecimento genérico de WordPress/Crocoblock. Toda afirmação sobre comportamento está ancorada em `arquivo:linha` do código realmente presente nos repositórios.

Versões analisadas:

| Plugin | Versão / caminho |
|---|---|
| JetFormBuilder | `core-plugins/jetformbuilder-v3.6.1.1/` |
| JetEngine | `core-plugins/jet-engine-v3.8.10.1/` |
| Elementor Pro | `core-plugins/elementor-pro-v4.1.2/` (usado só para ancorar a analogia do checkout) |
| Addons JFB (referência de padrão) | `secundary-plugins/jetformbuilder-addons/` |
| Precedente de real-time | `projeto-conversa/` (docs + plugin `conversa-chat`) |

**Onde o código diverge de qualquer documentação, o código prevalece.**

---

## 1. Arquivos analisados

### 1.1 JetFormBuilder — servidor

```
includes/blocks/types/form.php
includes/blocks/render/form-builder.php
includes/blocks/render/form-hidden-fields.php
includes/blocks/render/calculated-field-render.php
includes/blocks/block-helper.php
includes/blocks/module.php
includes/blocks/conditional-block/render-state.php
includes/blocks/conditional-block/condition-manager.php
includes/blocks/conditional-block/operators.php
includes/blocks/conditional-block/functions.php
includes/blocks/button-types/button-switch-state.php
includes/blocks/button-types/button-update.php
includes/form-handler.php
includes/form-response/types/ajax-response.php
includes/form-manager.php
includes/functions.php
includes/classes/http/http-tools.php
includes/presets/preset-manager.php
includes/presets/sources/preset-source-query-var.php
includes/generators/registry.php
includes/generators/base-v2.php
modules/actions-v2/call-hook/call-hook-action.php
modules/block-parsers/parser-context.php
modules/option-field/module.php
modules/option-field/rest-api/generator-update-endpoint.php
modules/option-field/rest-api/rest-api-controller.php
modules/shortcode/form-shortcode.php
modules/security/csrf/module.php
components/rest-api/rest-api-endpoint-base.php
compatibility/jet-engine/jet-engine.php
```

### 1.2 JetFormBuilder — cliente (bundles)

```
assets/build/frontend/main.js              (runtime: Observable, submit, inputs)
assets/build/frontend/calculated.field.js
assets/build/frontend/conditional.block.js
assets/build/frontend/dynamic.value.js
assets/build/frontend/multi.step.js
modules/jet-plugins/assets/build/index.js  (JetPlugins: hooks + (re)init de blocos)
modules/option-field/assets/build/auto-update.js
```

> Todos são bundles minificados. Para leitura foram reformatados em cópias de trabalho no scratchpad (`jfb-js/`); **nenhum arquivo dos repositórios foi tocado**. Por isso as citações desses bundles são por símbolo/string, não por linha do arquivo original.

### 1.3 JetEngine

```
includes/components/listings/ajax-handlers.php
includes/components/listings/render/listing-grid.php
includes/components/query-builder/listings/query.php
includes/modules/data-stores/inc/stores/factory.php
assets/js/frontend.js
```

### 1.4 Referências externas (padrão, não dependência)

```
secundary-plugins/jetformbuilder-addons/jet-form-builder-update-field-main/
    rest-api/endpoint.php, assets/js/frontend.js, README.md
secundary-plugins/jetformbuilder-addons/jet-form-builder-formless-actions-endpoints/
core-plugins/elementor-pro-v4.1.2/modules/woocommerce/module.php
projeto-conversa/conversa-chat/includes/class-conversa-chat-renderer.php
projeto-conversa/conversa-chat/assets/js/runtime.js
projeto-conversa/docs/02, 04, 06, 07
```

---

## 2. Classes e objetos analisados

**Servidor (JFB)**
`Jet_Form_Builder\Blocks\Types\Form` · `Blocks\Render\Form_Builder` · `Blocks\Render\Form_Hidden_Fields` · `Blocks\Render\Calculated_Field_Render` · `Blocks\Block_Helper` · `Blocks\Conditional_Block\Render_State` · `Blocks\Conditional_Block\Condition_Manager` · `Form_Handler` · `Form_Response\Types\Ajax_Response` · `Actions\Action_Handler` · `JFB_Modules\Actions_V2\Call_Hook\Call_Hook_Action` · `Presets\Preset_Manager` · `Presets\Sources\Preset_Source_Query_Var` · `Generators\Registry` · `Generators\Base_V2` · `JFB_Modules\Block_Parsers\Parser_Context` · `JFB_Modules\Option_Field\Rest_Api\Generator_Update_Endpoint` · `Classes\Http\Http_Tools` · `Live_Form`

**Cliente (JFB)**
`Observable` (exposto em `window.JetFormBuilderAbstract.Observable`) · `InputData` · `ReactiveVar` · `BaseSignal` · `CalculatedFormula` · a classe de submit AJAX (`status`, `submit`, `runSubmit`, `onSuccess`, `onFail`, `lastResponse`) · `window.JetPlugins`

**JetEngine**
`Jet_Engine_Listings_Ajax_Handlers` · render instance `listing-grid` (`posts_loop`) · `JetEngine` (objeto JS global)

---

## 3. O fluxo real, ponta a ponta

### 3.1 Render do formulário (servidor)

```
bloco Gutenberg  jet-forms/form-block
shortcode        [jet_fb_form form_id="X"]        modules/shortcode/form-shortcode.php:19,53
widget Elementor jet-engine-booking-form / jet-form-builder-form
        │
        ▼
Form::render_callback_field()                     includes/blocks/types/form.php:41
        │  envolve tudo em:
        │  <div class="jet-fb-form-block" data-is-block="jet-forms/form-block">   :43
        ▼
Form::render_form()                               :97
        ├─ filtro jet-form-builder/prevent-render-form                            :139
        └─ new Form_Builder( $form_id, $attrs )->render_form()                    :145
                │
                ├─ Live_Form::set_form_id()->set_specific_data_for_render()->setup_fields()
                │                                    render/form-builder.php:70-73
                ├─ start_form()                       :120
                │     data-form-id      = form_id     :142   ← seletor estável do form
                │     data-clear        = clear       :146   ← "Clear on submit" nativo
                │     action            = Http_Tools::get_form_action_url()  :140
                │     filtro jet-form-builder/after-start-form :171 ← onde CSRF injeta token
                ├─ Form_Hidden_Fields::render()       :77   → render-state, form_id, refer, post_id
                ├─ render_block() por bloco do form   :83
                └─ end_form()                         :176
```

Dois detalhes decisivos deste caminho:

1. **`render_styles()` já prevê execução dentro de AJAX/REST** — `render/form-builder.php:197-210`: se `wp_doing_ajax()` ou `REST_REQUEST`, os estilos do form saem **inline no HTML** em vez de enfileirados. Ou seja: renderizar um formulário dentro de uma requisição AJAX é um cenário **previsto pelo próprio código**.
2. **O wrapper carrega `data-is-block`** (`types/form.php:43`) — é exatamente o atributo que o runtime `JetPlugins` procura para (re)inicializar blocos. Ver §4.1.3.

### 3.2 Boot e reatividade no cliente

`assets/build/frontend/main.js` registra o inicializador do form em três portas:

```js
JetPlugins.bulkBlocksInit([{ block: "jet-forms.form-block", callback: Mn,
                             condition: () => "loading" !== document.readyState }]);

jQuery(window).on('elementor/frontend/init', () => {
  elementorFrontend.hooks.addAction('frontend/element_ready/jet-engine-booking-form.default', Mn);
  elementorFrontend.hooks.addAction('frontend/element_ready/jet-form-builder-form.default',  Mn);
});
```

E o inicializador (`Mn`) faz:

```js
const form = $scope[0].querySelector('form.jet-form-builder');
const observable = new Observable();
window.JetFormBuilder[ form.dataset.formId ] = observable;      // ← API pública, por form_id
jQuery(document).trigger('jet-form-builder/init',       [$scope, observable]);
observable.observe(form);
jQuery(document).trigger('jet-form-builder/after-init', [$scope, observable]);
```

**`window.JetFormBuilder[formId]` é o ponto de entrada oficial para qualquer coisa que queira falar com um formulário já renderizado.** API do `Observable` (main.js):

| Membro | O que faz |
|---|---|
| `rootNode` | o elemento `<form>` |
| `getInput(name)` | retorna o `InputData` do campo |
| `getInputs()` | todos os `InputData` |
| `watch(name, cb)` | observa um campo pelo nome (lança se não existir) |
| `observeInput(node, force)` | passa a observar um nó inserido depois; dispara `jet.fb.observe.input.manual` |
| `value.current` | objeto reativo `{ nome: valor }` do form inteiro |
| `getState()` | o input `_jfb_current_render_states` (estado de render) |
| `getSubmit()` | o handler de submit (tem `.status`, `.lastResponse`, `.submit()`) |
| `reQueryValues()` / `remove()` | re-lê valores / desliga os observadores |

Cada `InputData` tem `value` (um `ReactiveVar`: `.current` para ler/escrever, `.watch(cb)`, `.notify()`, `.setIfEmpty()`), `attrs`, `nodes`, `name`, `rawName`, `reporting`, `onClear()`.

**Escrever `input.value.current = x` propaga por todo o grafo reativo**: campos calculados recalculam, blocos condicionais reavaliam, dynamic values reaplicam. É o mesmo canal que o usuário digitando usa.

### 3.3 Submit (cliente → servidor → cliente)

```
submit AJAX (main.js)
  ├─ filtro JS  jet.fb.submit.ajax.promises   ← injetar promessas antes de enviar
  ├─ FormData(rootNode) + _jet_engine_booking_form_id
  └─ jQuery.ajax POST → JetFormBuilderSettings.ajaxurl
        │
        ▼  SERVIDOR — includes/form-handler.php
        process_ajax_form()                                    :230
        └─ process_form()                                      :240
             └─ try_send() → send_form()                       :274 / :307
                  ├─ request_handler->set_form_data()          :309
                  ├─ do_action('jet-form-builder/form-handler/before-send')   :311
                  ├─ jet_fb_events()->execute( Default_Process_Event )        :314
                  │     └─ executa as ACTIONS do form (Insert CCT, Call Hook, ...)
                  ├─ add_response_data( action_handler->response_data )       :316
                  └─ send_response() → do_action('.../after-send')            :354
                       └─ Ajax_Response::send() → wp_send_json($query_args)
                                              form-response/types/ajax-response.php:34
        │
        ▼  CLIENTE — onSuccess(response)
        this.lastResponse = response
        status === 'success'
          ? jQuery(document).trigger('jet-form-builder/ajax/on-success', [response, $form])
          : jQuery(document).trigger('jet-form-builder/ajax/processing-error', [...])
        response.redirect → navega | response.reload → window.location.reload()
        insertMessage(response.message)
```

**Consequência prática número um:** o JSON devolvido no submit é o `response_data` inteiro, mais `status` e `message`. Qualquer chave que o servidor colocar em `response_data` **chega ao cliente dentro do evento `jet-form-builder/ajax/on-success`**. Esse é o canal nativo para o Form [1] mandar carga útil para quem estiver ouvindo.

**Consequência prática número dois:** `response.reload` e `response.redirect` são justamente o que **não** queremos — a funcionalidade tem que viver no espaço entre "não faz nada" e "recarrega a página".

---

## 4. O arsenal nativo — o que já existe e pode ser dirigido

### 4.1 Cliente

#### 4.1.1 Sistema de hooks JS (`JetPlugins.hooks`)

`modules/jet-plugins/assets/build/index.js` implementa uma cópia da API `@wordpress/hooks` (`addAction`, `addFilter`, `doAction`, `applyFilters`, prioridades, namespaces). Os 64 hooks `jet.fb.*` registrados no JFB incluem, relevantes aqui:

| Hook | Tipo | Uso para a funcionalidade |
|---|---|---|
| `jet.fb.observe.before` / `jet.fb.observe.after` | action | receber o `Observable` de todo form que inicializa — **o gancho para "conectar" formulários no boot** |
| `jet.fb.input.makeReactive` | action | receber cada campo quando ele vira reativo (é o gancho que o addon Update Field usa) |
| `jet.fb.input.created` | action | receber cada `InputData` criado |
| `jet.fb.observe.input.manual` | action | campo observado manualmente depois do boot |
| `jet.fb.custom.formula.macro` | filter | **resolver macros `%ALGO::coisa%` em fórmulas** — ponto oficial para uma macro que leia outro formulário |
| `jet.fb.formula.node.exists` | filter | dizer ao motor de fórmula que um nome de campo "existe" mesmo não existindo no form local |
| `jet.fb.submit.ajax.promises` | filter | injetar promessas no submit AJAX (adiar/abortar/enriquecer) |
| `jet.fb.conditional.types` / `.checkers` | filter | novos tipos de condição para blocos condicionais |
| `jet.fb.dynamic.value.types` | filter | novos tipos de "dynamic value" |
| `jet.fb.inputs` / `jet.fb.signals` / `jet.fb.filters` | filter | novos tipos de input, sinais e filtros de valor |

Eventos jQuery (documento): `jet-form-builder/init`, `jet-form-builder/after-init`, `jet-form-builder/ajax/on-success`, `jet-form-builder/ajax/on-fail`, `jet-form-builder/ajax/processing-error`.

#### 4.1.2 O motor de fórmula (`CalculatedFormula`)

`main.js`, exposto em `window.JetFormBuilderAbstract.CalculatedFormula`:

- Sintaxe `%NOME%`, com filtros encadeados por `|`.
- `observeMacro()` resolve `%NOME%` como `this.root.getInput(NOME)` — **só o próprio formulário** — e registra `input.watch(() => this.setResult())`, o que faz a fórmula recalcular sozinha quando o campo muda.
- Quando o nome contém `::`, ele **não** procura campo: chama `applyFilters('jet.fb.custom.formula.macro', false, nome, args, this)`. Se o retorno for uma função, ela é reavaliada a cada cálculo.
- O quarto argumento entregue ao filtro é a própria instância da fórmula — que expõe `watchers`, `related` e `setResult`. **Isso significa que um handler desse filtro consegue registrar um watcher em um campo de outro formulário e forçar o recálculo.** É o caminho mais "declarativo" possível para conectar formulários pelo cliente.

#### 4.1.3 Re-hidratação de HTML injetado (`JetPlugins`)

`modules/jet-plugins/assets/build/index.js`:

```js
JetPlugins.init($scope, blocks)    // varre $scope.find('[data-is-block*="/"]') e chama initBlock
JetPlugins.initBlock(node, force)  // dispara a action jet-plugins.frontend.element-ready.{bloco}
                                   // e marca node.dataset.jetInited = true
JetPlugins.registerBlockHandlers({ block, callback, condition })
```

`initBlock` só age se `dataset.jetInited === undefined` (ou se a `condition` registrada permitir). Como o HTML vindo do servidor nasce sem `data-jet-inited`, **injetar o HTML de um formulário e chamar `JetPlugins.init(jQuery(container))` reinicializa o formulário pelo caminho oficial**, criando um `Observable` novo e disparando `jet-form-builder/init` / `after-init`.

Esta é a peça que torna viável trocar um formulário inteiro sem reload — e sem nenhum "renderer espelho".

#### 4.1.4 Reatividade declarativa já pronta dentro de um form

- **Calculated Field** — `includes/blocks/render/calculated-field-render.php:42-66`. Macros na fórmula: `%FIELD::nome%` vira `%nome%` (resolvido no cliente); `%META::chave%` é resolvido **no servidor** com `get_post_meta()`; qualquer outro prefixo cai no filtro `jet-engine/calculated-data/{macro}` — **ponto de extensão server-side de macro de fórmula**.
- **Dynamic Value** — `assets/build/frontend/dynamic.value.js`: um campo recebe valor de uma fórmula, com `conditions`, `frequency` (`always` / `on_change` / `once`) e `set_on_empty`. Lê `data-value` / `data-dynamic-value`.
- **Conditional Block** — `conditional.block.js` + `includes/blocks/conditional-block/`: mostra/esconde/aplica classe conforme condições. Extensível por três filtros PHP: `jet-form-builder/conditional-block/types` (`condition-manager.php:46`), `.../operators` (`operators.php:62`), `.../functions` (`functions.php:28`).
- **Render States** — `includes/blocks/conditional-block/render-state.php`. O form carrega um hidden `_jfb_current_render_states[]` (`:32`); condições podem usar o operador `render_state`; um botão `.jet-form-builder__button-switch-state` troca o estado **no cliente** (`observable.getState().value.current = data-switch-on`, ver `button-switch-state.php:28-54`). O estado também pode vir da URL: `?jfb[{form_id}][state]=NOME` (`render-state.php:232`). É um mecanismo nativo de "o formulário muda de cara sem reload".

#### 4.1.5 Auto Update nativo (o mais próximo do que queremos)

O JFB 3.6.1.1 **já tem, no core**, um mecanismo de "campo reage a outro campo indo ao servidor":

- Endpoint: `POST /wp-json/jet-form-builder/v1/generator-update`
  (`components/rest-api/rest-api-endpoint-base.php:15-16` define o namespace; `modules/option-field/rest-api/generator-update-endpoint.php:28-30` define o `rest_base`).
- Corpo: `{ form_id, field_name, generator_id, context: { campo: valor, ... } }` (`generator-update-endpoint.php:59-80`).
- Cliente: `modules/option-field/assets/build/auto-update.js`, dirigido por data-attributes no bloco: `data-jfb-auto-update`, `data-listen-to`, `data-listen-to-multiple`, `data-generator-id`, `data-require-all-filled`, `data-empty-context-action`, `data-update-on-button`, `data-cache-timeout`, `data-form-id`.
- Segurança modelar: os `block_attrs` são lidos **do post salvo**, nunca do cliente, e o `generator_id` do request é conferido contra o salvo (`:104-122`). Contexto sanitizado em `:251-268`.
- O contexto fica disponível ao servidor em `$GLOBALS['jfb_generator_context']` (`:129`) e, por compatibilidade, em `$_REQUEST['jfb_update_related_{campo}']` (`:135`), sempre limpos no `finally` (`:164-176`).
- Extensão: registrar um generator próprio via filtro `jet-form-builder/forms/options-generators` (`includes/form-manager.php:36`), estendendo `Generators\Base_V2` e implementando `supports_auto_update()` (`base-v2.php:196`), `get_auto_update_context_fields()` (`:206`) e `generate_with_context()` (`:280`).

**Limite comprovado:** o `context` enviado é montado a partir dos campos **do próprio formulário**. Não há nada no fluxo que aceite valores de outro `form_id`.

### 4.2 Servidor

| Necessidade | Mecanismo nativo | Evidência |
|---|---|---|
| Renderizar um formulário fora do fluxo de página | `jet_fb_render_form()` / shortcode `[jet_fb_form]` / `Form::render_callback_field()` | `includes/functions.php:25`; `modules/shortcode/form-shortcode.php:19,53`; `includes/blocks/types/form.php:41` |
| Substituir a renderização de um form | filtro `jet-form-builder/prevent-render-form` | `includes/blocks/types/form.php:139` |
| Intervir antes do form montar | filtro `jet-form-builder/pre-render/{form_id}` | `render/form-builder.php:112` |
| Injetar markup no início/fim do form | `jet-form-builder/after-start-form`, `before-end-form`, `after-end-form` | `render/form-builder.php:171,183,192` |
| Corrigir URLs em render fora de página | `jet-form-builder/form-action-url`, `jet-form-builder/form-refer-url` | `classes/http/http-tools.php:100,120` |
| Rodar código próprio no submit e **devolver dados ao cliente** | action **Call Hook** → `do_action('jet-form-builder/custom-action/{nome}')` e `apply_filters('jet-form-builder/custom-filter/{nome}')`, cujo retorno vai para `response_data['hook_result']` | `modules/actions-v2/call-hook/call-hook-action.php:50-67` |
| Acrescentar qualquer carga à resposta AJAX | `jet_fb_handler()->add_response_data([...])` ou `$handler->response_data[...]` dentro de uma action | `includes/form-handler.php:327`; exemplos em `actions-v2/insert-post/properties/base-post-action.php:61-65` |
| Reagir a todo submit, antes/depois das actions | `jet-form-builder/form-handler/before-send` (`:311`), `.../after-send` (`:354`) | `includes/form-handler.php` |
| Montar o contexto de um formulário sem submeter | `Block_Helper::get_blocks_by_post($form_id)` + `jet_fb_handler()->set_form_id()` + `jet_fb_context()->set_request($valores)->apply($blocks)` | `includes/blocks/block-helper.php:186`; `includes/functions.php:104`; `modules/block-parsers/parser-context.php:58,285` |
| Pré-preencher campos a partir de contexto externo | Presets (`post`, `user`, `term`, `query_var`) | `includes/presets/sources/` — `preset-source-query-var.php` lê `$_GET` |
| Gerar opções de campo dinamicamente | Generators + `Registry::generate()` | `includes/generators/registry.php:265` |

### 4.3 JetEngine — o padrão "fragment refresh" já existe

Este é o achado que mais aproxima o ecossistema do modelo de checkout.

**Endpoint AJAX de listagem** — `includes/components/listings/ajax-handlers.php`
- action `jet_engine_ajax` (`:28-29`), handlers permitidos `listing_load_more` e `get_listing` (`:105-115`)
- `listing_load_more()` (`:328`) roda `$render_instance->posts_loop(...)` (`:459`) — o **mesmo** loop do grid
- `get_listing()` (`:479`) re-renderiza o listing inteiro e devolve `html` (`:536-540`)
- `maybe_add_enqueue_assets_data()` (`:577`) devolve **scripts e styles** que o HTML novo precisa, já como tags prontas
- filtro de saída: `jet-engine/ajax/get_listing/response`
- a query vai **assinada** (HMAC com as chaves do site) e é validada em `get_validated_query_from_request()` (`:195`) — não dá para forjar query pelo cliente

**Cliente** — `assets/js/frontend.js`
- `JetEngine.ajaxGetListing(options, done, fail)` (`:1277`)
- `JetEngine.enqueueAssetsFromResponse(response)` (`:2578`) carrega os assets devolvidos
- `JetEngine.initElementsHandlers($html)` (`:2473`) religa os widgets Elementor do HTML novo
- **`response.data.fragments`** (`:1403-1411`): um mapa `{ seletor CSS: html }`; para cada seletor presente na página, o JetEngine faz `$(seletor).html(valor)`. O mesmo padrão aparece em mais dois pontos do arquivo (`:478`, `:684`)
- evento `jet-engine/listing/ajax-get-listing/done` (`:1414`)
- `JetEngine.dataStoreSyncListings(args)` (`:533`) — ao mudar um Data Store, re-renderiza listings alvo via `get_listing`

**Quem produz fragments hoje:**
- Data Stores — filtro `jet-engine/data-stores/ajax-store-fragments` (`includes/modules/data-stores/inc/stores/factory.php:189,238`)
- Query Builder — contadores de itens (`includes/components/query-builder/listings/query.php:145-159`)

**O grid publica no DOM tudo que é preciso para pedir sua própria atualização**: `data-nav` (com `query` assinada e `widget_settings`), `data-listing-id`, `data-query-id`, `data-page` (`render/listing-grid.php:1519`), liberados pelo filtro `jet-engine/listing/grid/add-query-data` (`:1327`).

---

## 5. A analogia do checkout, ancorada em código presente

O WooCommerce em si não está nos repositórios, mas o **mecanismo** que o autor descreve está, no Elementor Pro:

- `core-plugins/elementor-pro-v4.1.2/assets/js/woocommerce-checkout-page.…bundle.js`: ao aplicar cupom, o código dispara `applied_coupon_in_checkout` e em seguida `update_checkout` no `$body`; e escuta `updated_checkout` para reaplicar seus próprios ajustes de UI. O comentário do arquivo cita explicitamente as chamadas AJAX `update_order_review` e `update_cart`.
- `core-plugins/elementor-pro-v4.1.2/modules/woocommerce/module.php:327-348`: `menu_cart_fragments()` monta `['fragments' => [seletor => html]]` e responde com `wp_send_json`. `get_fragments_handler()` (`:407-417`) constrói o mapa `seletor → html`.

**Tradução do modelo:**

| Checkout WooCommerce | Equivalente disponível aqui |
|---|---|
| `applied_coupon` (algo mudou) | `jet-form-builder/ajax/on-success` (`form-handler.php` → `main.js`) ou `input.value.watch()` |
| `update_checkout` (pedido de atualização) | chamada a um endpoint (REST do JFB, `jet_engine_ajax`, ou próprio) |
| `update_order_review` (servidor recalcula e devolve HTML) | `posts_loop()` / `get_listing` / `Form_Builder::render_form()` / `Generator_Update_Endpoint` |
| `fragments` (mapa seletor → html) | `response.data.fragments` do JetEngine (`frontend.js:1403`) |
| `updated_checkout` (avisa a UI) | `jet-engine/listing/ajax-get-listing/done`, ou evento próprio |

Ou seja: **o padrão que o autor quer é exatamente o padrão que o JetEngine já implementa** — só que hoje ele está cabeado a listings e data stores, não a formulários.

---

## 6. O que NÃO existe — lacunas comprovadas

Declaração explícita, conforme a Regra Nº 5:

1. **Não localizei, no código analisado, nenhum mecanismo nativo de comunicação entre dois formulários distintos.**
   - No cliente, `window.JetFormBuilder[formId]` é populado por form (`main.js`), e `CalculatedFormula.observeMacro()` resolve nomes de campo apenas em `this.root` — o próprio formulário.
   - No servidor, os Presets têm fontes `post`, `user`, `term`, `query_var` e (via JetEngine) `options-page`; **não há fonte "outro formulário"**.
   - O `Generator_Update_Endpoint` monta `context` a partir dos campos do próprio form.
   - Busca por `JetFormBuilder[...]` nos bundles do plugin retorna apenas o registro e dois acessos indexados; nenhuma varredura cross-form.

2. **Não localizei action, filtro ou endpoint nativo que re-renderize um formulário JetFormBuilder por AJAX.** O caminho existe (o render funciona em AJAX/REST, `form-builder.php:198`), mas **não há um endpoint pronto que o exponha** — ao contrário do que ocorre com listings (`get_listing`).

3. **Não localizei suporte a `fragments` no JetFormBuilder.** O `Ajax_Response::send()` (`ajax-response.php:34`) só faz `wp_send_json($query_args)`; nenhum consumidor de `fragments` existe no cliente do JFB. O mecanismo de fragments é do JetEngine (`frontend.js:1403`).

4. **`window.JetFormBuilder` é indexado por `form_id`.** Duas instâncias do mesmo formulário na mesma página **sobrescrevem uma à outra** no registro global (`main.js`: `window.JetFormBuilder[e.dataset.formId] = n`). Qualquer desenho que dependa de "achar o form pelo id" precisa lidar com isso.

Portanto: **a funcionalidade é uma camada nova.** O que já existe são todas as peças para construí-la sem reinventar render nem reatividade.

---

## 7. Estratégias possíveis (todas ancoradas no que foi encontrado)

As cinco não são exclusivas: a solução provável combina A + (B ou C) + E.

### A — Ponte reativa no cliente (sem servidor)

**Mecanismo:** no `jet.fb.observe.after` (ou no `jet-form-builder/after-init`), registrar os `Observable` num registro próprio; ligar `observableA.getInput('x').value.watch(...)` a `observableB.getInput('y').value.current = ...`.

**Também via fórmula, de forma declarativa:** um handler de `jet.fb.custom.formula.macro` que entenda `%FORM::56386.cupom%`, devolva uma função de leitura e registre o watcher no input remoto usando a instância de fórmula recebida como quarto argumento.

- **Prós:** zero requisições; usa o grafo reativo nativo; campos calculados e blocos condicionais do form [2] reagem sozinhos; funciona nos dois sentidos.
- **Contras:** só transporta o que o cliente já sabe. Não valida, não consulta banco, não recalcula preço com regra de negócio.
- **Quando:** espelhar valores, habilitar/desabilitar seções, alimentar fórmulas.

### B — Re-render parcial de campo pelo servidor (padrão Auto Update / Update Field)

**Mecanismo:** endpoint REST que recebe `{form_id_alvo, field_name, context}`, remonta o contexto com `Block_Helper::get_blocks_by_post()` + `jet_fb_context()->set_request()->apply()`, renderiza **apenas** aquele campo e devolve valor/opções/HTML. O cliente aplica via `input.value.current = ...` ou substituindo o HTML do grupo de campos.

**Precedentes exatos:** o nativo `Generator_Update_Endpoint` (`modules/option-field/rest-api/generator-update-endpoint.php`) e o addon `jet-form-builder-update-field-main/rest-api/endpoint.php` — que faz literalmente esse setup de contexto nas linhas 74-77 e devolve `{type: value|options|block, value}`.

- **Prós:** o campo continua sendo renderizado pelo JFB; carga mínima; padrão já validado pelo próprio time do plugin; a segurança já tem modelo pronto (attrs lidos do post, nunca do cliente).
- **Contras:** não cobre "mudou o layout inteiro do form [2]"; cada tipo de campo precisa de tratamento no cliente.
- **Quando:** o form [2] precisa recalcular um valor/opções com regra de negócio no servidor.

### C — Re-render do formulário inteiro e re-hidratação nativa

**Mecanismo:**
1. endpoint devolve o HTML de `Form::render_callback_field(['form_id' => 2, ...])` (ou `do_shortcode('[jet_fb_form form_id="2"]')`);
2. o cliente descarta o `Observable` antigo (`observable.remove()`), substitui o `<div class="jet-fb-form-block">`, e chama `JetPlugins.init(jQuery(container))`;
3. o JFB reinicializa o form pelo caminho oficial e dispara `jet-form-builder/after-init`.

- **Prós:** é o análogo direto do `update_order_review`; o formulário [2] volta exatamente como o autor o desenhou no editor — **nenhum contrato implícito de layout com o JS**, que é justamente o erro que o `projeto-conversa` documentou em `docs/04`.
- **Contras:** perde estado não submetido do form [2] (foco, valores digitados, scroll) se não for preservado deliberadamente; exige cuidado com `refer`/`action`/nonce (ver §8).
- **Quando:** o form [2] muda de estrutura, não só de valor.

> Nota: parte disso pode ser resolvido **sem** re-render, usando **Render States** (§4.1.4) — trocar o estado no cliente já revela/esconde blocos condicionais desenhados no editor.

### D — Fragments no estilo JetEngine

**Mecanismo:** a resposta (do submit ou de um endpoint próprio) traz `fragments: { seletor: html }` e o cliente aplica. Para conteúdo renderizado por JetEngine (Listing, Dynamic Field), pode-se reaproveitar literalmente `JetEngine.ajaxGetListing` + `enqueueAssetsFromResponse` + `initElementsHandlers`.

- **Prós:** padrão já existente no ecossistema (`frontend.js:1403`); permite atualizar num golpe só o form [2] **e** a área de "informação x" ao redor dele.
- **Contras:** o consumidor de fragments é do JetEngine; para uma resposta do JFB seria preciso um consumidor próprio (pequeno, mas novo).
- **Quando:** "informação x" é conteúdo do JetEngine (listing/campo dinâmico), não campo de formulário.

### E — Carona na resposta do submit

**Mecanismo:** o form [1] usa a action **Call Hook** (ou uma action própria) para rodar a regra de negócio; o retorno de `apply_filters('jet-form-builder/custom-filter/{nome}')` chega ao cliente em `response.hook_result` (`call-hook-action.php:62`). Alternativamente, `jet_fb_handler()->add_response_data([...])` em `form-handler/before-send` / dentro de uma action. No cliente, `jet-form-builder/ajax/on-success` recebe o payload e alimenta A, B, C ou D.

- **Prós:** zero requisições extras — o dado já vem na resposta do envio; 100% nativo, inclusive configurável pela UI (Call Hook é uma action de formulário).
- **Contras:** só cobre o gatilho "submit". Não cobre "o usuário digitou algo no form [2]".
- **Quando:** sempre que o gatilho for um envio. É o par natural do que o autor descreveu como "atualizo o formulário [1] → o [2] responde".

### Combinação recomendada para o caso descrito

```
[1] submit  →  action Call Hook / action própria calcula
            →  response_data (E)
            →  on-success no cliente
            →  aplica em [2]:  valor simples  → A (input.value.current)
                               recálculo real → B (endpoint de campo)
                               mudou a cara   → Render State, ou C (re-render + JetPlugins.init)
                               conteúdo Jet   → D (fragments / ajaxGetListing)

[2] alterado (sem submit) →  watch no input (A)  →  mesmo pipeline na direção inversa
```

---

## 8. Riscos e pontos de atenção — todos comprovados no código

1. **`refer` e `action` em render fora da página.**
   `Form_Hidden_Fields::render()` grava o campo `refer` com `Http_Tools::get_form_refer_url()`, que usa `global $wp` e `$_SERVER['QUERY_STRING']` (`http-tools.php:105-120`); `start_form()` monta o `action` com `get_form_action_url()` (`:83-100`). Renderizado dentro de um endpoint, **os dois apontariam para a URL do endpoint**, e `Form_Handler::is_process()` (`form-handler.php:174`) depende do `refer`.
   **Mitigação nativa:** os filtros `jet-form-builder/form-refer-url` e `jet-form-builder/form-action-url` existem exatamente para isso.

2. **Assets do formulário re-renderizado.**
   `Blocks\Module::enqueue_frontend_assets()` só chama `wp_enqueue_script()` — inócuo dentro de AJAX. O CSS está resolvido (inline, `form-builder.php:198-210`), mas se o form [2] usar um script que a página ainda não carregou (`calculated.field.js`, `conditional.block.js`, `media.field.js`), **ele não virá**.
   **Mitigações:** pré-enfileirar na página, ou copiar o padrão do JetEngine de devolver scripts/styles na resposta (`ajax-handlers.php:577` + `frontend.js:2578`).

3. **Nonce, CSRF e honeypot.** São injetados por hooks de render (ex.: CSRF em `jet-form-builder/after-start-form`, `modules/security/csrf/module.php:50`). Um re-render pelo caminho oficial os reproduz; um re-render "manual" que pule `start_form()` os perderia. **Passar sempre por `Form_Builder`.**

4. **Colisão de `form_id` no registro global.** `window.JetFormBuilder[formId]` sobrescreve (§6.4). Dois formulários iguais na página, ou o mesmo formulário dentro de um listing repetido, quebram a resolução por id. Resolver por nó DOM (`rootNode`) sempre que possível.

5. **Laço de eco na ligação bidirecional.** O desenho pedido é `[1] → [2] → [1]`. Como escrever em `input.value.current` dispara os mesmos watchers que a digitação do usuário, uma ponte ingênua entra em laço. Precisa de guarda de origem (marcar a atualização como "vinda da ponte" e ignorá-la do outro lado) e/ou comparação de valor antes de escrever — o `dynamic.value.js` já usa essa ideia com `frequency: 'on_change'` e `prevResult`.

6. **Corridas de requisição.** O addon Update Field trata isso com `AbortController` por campo e cache com timeout (`assets/js/frontend.js:318-360`); o `auto-update.js` nativo tem `data-cache-timeout`. Qualquer endpoint novo deve seguir o mesmo padrão.

7. **`response.reload` / `response.redirect`.** Se o form [1] estiver configurado com "Redirect to Page" ou o processamento devolver `reload`, o cliente navega (`main.js`, `onSuccess`) e a funcionalidade some. **Requisito de configuração: form [1] em Submit Type = AJAX, sem redirect/reload.** É o mesmo requisito que o `projeto-conversa` registrou em `docs/02, §2.6`.

8. **Confiança no cliente.** O modelo nativo (`generator-update-endpoint.php:104-122`) é explícito: **carregar os atributos do bloco do post salvo, nunca do request**, e conferir o identificador enviado contra o salvo. Qualquer endpoint desta funcionalidade deve seguir isso, e validar capability/contexto de quem pode ver o form [2].

9. **Não recriar HTML de card/campo no cliente.** O `projeto-conversa` documenta em `docs/04, §4.3-4.4` o custo real dessa escolha: seletores Elementor hard-coded no JS, que quebram a cada edição de layout. A regra herdada é: **quem sabe desenhar é o template; o cliente só insere e re-hidrata.**

---

## 9. Precedente validado no `projeto-conversa`

O projeto já resolveu, para *listings*, o problema equivalente ao nosso — vale como prova de que o caminho "servidor renderiza pelo template real, cliente insere e re-hidrata" funciona em produção:

- `conversa-chat/includes/class-conversa-chat-renderer.php` — renderiza itens novos com `jet_engine()->listings->get_render_instance('listing-grid', $settings)->posts_loop(...)`, isto é, **o mesmo pipeline do `listing_load_more`**, com o contexto global (`$post`, `set_listing_by_id`) montado antes.
- `conversa-chat/assets/js/runtime.js` — `fullRefresh()` monta um POST para `jet_engine_ajax` / `get_listing` usando o `data-nav` que o próprio grid publicou no DOM, troca o grid e chama `JetEngine.enqueueAssetsFromResponse()` + `initElementsHandlers()`.
- `docs/07` registra o antes/depois: o que era "mirror renderer" (clonar card e trocar textos por seletor Elementor) virou "pipeline nativo".

**A lição transferível, em uma frase:** a versão que lutava contra os plugins tinha 6 snippets e um contrato implícito com IDs de elemento; a versão que dirigiu as capacidades nativas tem 2 endpoints e nenhum seletor de layout no JS.

---

## 10. Decisões em aberto (precisam da sua definição antes de desenhar a implementação)

1. **Gatilho no form [2]:** só `on-success` do form [1] (submit), ou também mudança de campo sem submit? Isso decide se precisamos de endpoint próprio (B/C) ou só da carona no submit (E).
2. **Granularidade da atualização:** um campo, um grupo de campos, o formulário inteiro, ou "formulário + área de conteúdo ao redor"? Decide entre B, C e D.
3. **Preservar o que o usuário digitou** no form [2] quando ele for atualizado? (No checkout, o Woo preserva; é trabalho extra em C.)
4. **Escopo do pareamento:** os formulários se conhecem por configuração no editor (um campo "escutar formulário X"), por atributo no HTML, ou por um registro programático via filtro?
5. **Bidirecional de verdade** ou "um manda, o outro responde e devolve confirmação"? Muda a complexidade da guarda de eco (§8.5).
6. **Onde mora a regra de negócio** (o "cupom"): numa action Call Hook, numa action própria, numa Query do JetEngine, ou num generator `Base_V2`?
7. **Formato de distribuição:** plugin próprio (como o `conversa-chat`) ou addon no padrão dos `secundary-plugins`? O `projeto-conversa/docs/06, §6.3` já argumentou a favor de plugin versionado, com liga/desliga e dependências declaradas.

---

## Anexo A — Referência rápida de ganchos

**PHP (JetFormBuilder)**
```
jet-form-builder/prevent-render-form            substituir o render do form
jet-form-builder/pre-render/{form_id}           abortar/alterar antes de montar
jet-form-builder/before-start-form              markup antes do <form>
jet-form-builder/after-start-form               markup após o <form> (CSRF vive aqui)
jet-form-builder/before-end-form
jet-form-builder/after-end-form
jet-form-builder/form-action-url                corrigir action em render AJAX
jet-form-builder/form-refer-url                 corrigir refer em render AJAX
jet-form-builder/form-handler/before-send       antes das actions (validação/anti-flood)
jet-form-builder/form-handler/after-send        depois de tudo
jet-form-builder/custom-action/{nome}           action do Call Hook
jet-form-builder/custom-filter/{nome}           retorno vai para response_data['hook_result']
jet-form-builder/forms/options-generators       registrar generator (auto-update)
jet-form-builder/render-states                  registrar render states
jet-form-builder/conditional-block/types        novos tipos de condição
jet-form-builder/conditional-block/operators    novos operadores
jet-form-builder/conditional-block/functions    novas funções (show/hide/...)
jet-engine/calculated-data/{macro}              macro de fórmula resolvida no servidor
```

**PHP (JetEngine)**
```
jet-engine/ajax/get_listing/response            acrescentar html/fragments/assets
jet-engine/listing/grid/add-query-data          publicar data-nav no grid
jet-engine/data-stores/ajax-store-fragments     fragments de data store
jet-engine/ajax-handlers/before-call-handler
```

**JS (JetPlugins.hooks)**
```
jet.fb.observe.before / jet.fb.observe.after    recebe o Observable de cada form
jet.fb.input.makeReactive                       recebe cada campo ao virar reativo
jet.fb.input.created / jet.fb.observe.input.manual
jet.fb.custom.formula.macro                     resolver %ALGO::coisa% em fórmulas
jet.fb.formula.node.exists                      declarar campo "existente"
jet.fb.submit.ajax.promises                     interceptar o submit AJAX
jet.fb.dynamic.value.types / jet.fb.conditional.types / jet.fb.inputs / jet.fb.signals
```

**Eventos jQuery**
```
jet-form-builder/init                    ($scope, observable)
jet-form-builder/after-init              ($scope, observable)
jet-form-builder/ajax/on-success         (response, $form)   ← response = response_data + status + message
jet-form-builder/ajax/on-fail            (xhr, status, err, $form)
jet-form-builder/ajax/processing-error   (response, $form)
jet-engine/listing/ajax-get-listing/done ($html, options)
```

**APIs globais**
```
window.JetFormBuilder[formId]        Observable do form
window.JetFormBuilderAbstract        Observable, InputData, ReactiveVar, CalculatedFormula, ...
window.JetFormBuilderFunctions       toHTML, validateInputs, populateInputs, queryByAttrValue, ...
window.JetFormBuilderSettings        ajaxurl, devmode, builtInStates, replaceAttrs, auto_focus
window.JetPlugins                    hooks, init($scope), initBlock(node), registerBlockHandlers()
window.JetEngine                     ajaxGetListing, initElementsHandlers, enqueueAssetsFromResponse
```

**Endpoints nativos utilizáveis**
```
POST /wp-json/jet-form-builder/v1/generator-update    { form_id, field_name, generator_id, context }
POST admin-ajax.php  action=jet_engine_ajax  handler=get_listing | listing_load_more
POST JetFormBuilderSettings.ajaxurl                   submit do formulário (FormData)
```
