<!--
//
// Copyright © 2022 Hardcore Engineering Inc.
//
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//
// See the License for the specific language governing permissions and
// limitations under the License.
//
-->
<script lang="ts">
  import { Plugin, PluginKey, TextSelection, Transaction } from 'prosemirror-state'
  import { DecorationSet } from 'prosemirror-view'
  import { getContext, createEventDispatcher, onDestroy, onMount } from 'svelte'
  import { WebsocketProvider } from 'y-websocket'
  import * as Y from 'yjs'
  import { AnyExtension, Editor, Extension, HTMLContent, getMarkRange } from '@tiptap/core'
  import Collaboration from '@tiptap/extension-collaboration'
  import CollaborationCursor from '@tiptap/extension-collaboration-cursor'
  import Placeholder from '@tiptap/extension-placeholder'
  import BubbleMenu from '@tiptap/extension-bubble-menu'
  import { generateId, getCurrentAccount, Markup } from '@hcengineering/core'
  import { IntlString, translate } from '@hcengineering/platform'
  import { Component, getPlatformColorForText, IconObjects, IconSize, themeStore } from '@hcengineering/ui'

  import textEditorPlugin from '../plugin'
  import { CollaborationIds, TextFormatCategory, TextNodeAction } from '../types'

  import { calculateDecorations } from './diff/decorations'
  import { defaultExtensions } from './extensions'
  import { NodeHighlightExtension, NodeHighlightType } from './extension/nodeHighlight'
  import TextEditorStyleToolbar from './TextEditorStyleToolbar.svelte'

  import StyleButton from './StyleButton.svelte'
  import { NodeUuidExtension } from './extension/nodeUuid'

  export let documentId: string
  export let readonly = false
  export let visible = true

  export let token: string
  export let collaboratorURL: string

  export let isFormatting = true
  export let buttonSize: IconSize = 'small'
  export let focusable: boolean = false
  export let placeholder: IntlString = textEditorPlugin.string.EditorPlaceholder
  export let initialContentId: string | undefined = undefined
  // export let suggestMode = false
  export let comparedVersion: Markup | ArrayBuffer | undefined = undefined

  export let field: string | undefined = undefined

  export let autoOverflow = false
  export let initialContent: string | undefined = undefined
  export let textNodeActions: TextNodeAction[] = []
  export let extensions: AnyExtension[] = []
  export let isNodeHighlightModeOn: boolean = false
  export let onNodeHighlightType: (uuid: string) => NodeHighlightType = () => NodeHighlightType.WARNING

  const ydoc = (getContext(CollaborationIds.Doc) as Y.Doc | undefined) ?? new Y.Doc()
  const contextProvider = getContext(CollaborationIds.Provider) as WebsocketProvider | undefined
  const wsProvider =
    contextProvider ??
    new WebsocketProvider(collaboratorURL, documentId, ydoc, {
      params: {
        token,
        documentId,
        initialContentId: initialContentId ?? ''
      }
    })

  if (contextProvider === undefined) {
    wsProvider?.on('status', (event: any) => {
      console.log(documentId, event.status) // logs "connected" or "disconnected"
    })
  }

  const currentUser = getCurrentAccount()

  let currentTextNodeAction: TextNodeAction | undefined | null
  let selectedNodeUuid: string | null | undefined
  let textNodeActionMenuElement: HTMLElement
  let element: HTMLElement
  let editor: Editor

  let placeHolderStr: string = ''

  $: ph = translate(placeholder, {}, $themeStore.language).then((r) => {
    placeHolderStr = r
  })

  const dispatch = createEventDispatcher()

  export function getHTML (): string | undefined {
    if (editor) {
      return editor.getHTML()
    }
  }

  export function selectNode (uuid: string) {
    if (!editor) {
      return
    }

    const { doc, schema, tr } = editor.view.state
    let foundNode = false
    doc.descendants((node, pos) => {
      if (foundNode) {
        return false
      }

      const nodeUuidMark = node.marks.find(
        (mark) => mark.type.name === NodeUuidExtension.name && mark.attrs[NodeUuidExtension.name] === uuid
      )

      if (!nodeUuidMark) {
        return
      }

      foundNode = true

      // the first pos does not contain the mark, so we need to add 1 (pos + 1) to get the correct range
      const range = getMarkRange(doc.resolve(pos + 1), schema.marks[NodeUuidExtension.name])

      if (!range) {
        return false
      }

      const [$start, $end] = [doc.resolve(range.from), doc.resolve(range.to)]

      editor.view.dispatch(tr.setSelection(new TextSelection($start, $end)))
      needFocus = true
    })
  }

  let needFocus = false

  let focused = false
  export function focus (): void {
    needFocus = true
  }

  $: if (editor && needFocus) {
    if (!focused) {
      editor.commands.focus()
    }
    needFocus = false
  }

  $: if (editor !== undefined) {
    editor.setEditable(!readonly)
  }

  // const isSuggestMode = () => suggestMode

  let _decoration = DecorationSet.empty
  let oldContent = ''

  function updateEditor (editor?: Editor, field?: string, comparedVersion?: Markup | ArrayBuffer): void {
    const r = calculateDecorations(editor, oldContent, field, comparedVersion)
    if (r !== undefined) {
      oldContent = r.oldContent
      _decoration = r.decorations
    }
  }

  const updateDecorations = () => {
    if (editor && editor.schema) {
      updateEditor(editor, field, comparedVersion)
    }
  }

  const getNodeUuid = () => {
    if (editor.view.state.selection.empty) {
      return null
    }
    if (!selectedNodeUuid) {
      selectedNodeUuid = generateId()
      editor.chain().setUuid(selectedNodeUuid!).run()
    }

    return selectedNodeUuid
  }

  const DecorationExtension = Extension.create({
    addProseMirrorPlugins () {
      return [
        new Plugin({
          key: new PluginKey('diffs'),
          props: {
            decorations (state) {
              updateDecorations()
              if (showDiff) {
                return _decoration
              }
              return undefined
            }
          }
        })
      ]
    }
  })

  $: updateEditor(editor, field, comparedVersion)

  onMount(() => {
    ph.then(() => {
      editor = new Editor({
        element,
        editable: true,
        extensions: [
          ...defaultExtensions,
          Placeholder.configure({ placeholder: placeHolderStr }),

          Collaboration.configure({
            document: ydoc,
            field
          }),
          CollaborationCursor.configure({
            provider: wsProvider,
            user: {
              name: currentUser.email,
              color: getPlatformColorForText(currentUser.email, $themeStore.dark)
            }
          }),
          DecorationExtension,
          NodeHighlightExtension.configure({
            isHighlightModeOn: () => isNodeHighlightModeOn,
            getNodeHighlightType: onNodeHighlightType,
            onNodeSelected: (uuid: string | null) => {
              if (selectedNodeUuid !== uuid) {
                selectedNodeUuid = uuid
                dispatch('node-selected', selectedNodeUuid)
              }
            }
          }),
          BubbleMenu.configure({
            pluginKey: 'text-node-action-menu',
            element: textNodeActionMenuElement,
            tippyOptions: {
              maxWidth: '38rem',
              onClickOutside: () => {
                currentTextNodeAction = undefined
              }
            },
            shouldShow: (editor) => {
              if (!editor) {
                return false
              }

              return !!currentTextNodeAction
            }
          }),
          ...extensions
        ],
        onTransaction: () => {
          // force re-render so `editor.isActive` works as expected
          editor = editor
        },
        onBlur: ({ event }) => {
          focused = false
          dispatch('blur', event)
        },
        onFocus: () => {
          focused = true
        },
        onUpdate: (op: { editor: Editor; transaction: Transaction }) => {
          dispatch('content', editor.getHTML())
        }
      })

      if (initialContent) {
        editor.commands.insertContent(initialContent as HTMLContent)
      }
    })
  })

  onDestroy(() => {
    if (editor) {
      try {
        editor.destroy()
      } catch (err: any) {}
      if (contextProvider === undefined) {
        wsProvider.disconnect()
      }
    }
  })

  let showDiff = true
</script>

<div class="actionPanel" bind:this={textNodeActionMenuElement}>
  {#if !!currentTextNodeAction}
    <Component
      is={currentTextNodeAction.panel}
      props={{
        documentId,
        field,
        editor,
        action: currentTextNodeAction,
        disabled: editor.view.state.selection.empty,
        onNodeUuid: getNodeUuid
      }}
      on:close={() => {
        currentTextNodeAction = undefined
      }}
    />
  {/if}
</div>
{#if visible}
  <div class="ref-container" class:autoOverflow>
    {#if isFormatting && !readonly}
      <div class="formatPanelRef formatPanel flex-between clear-mins">
        <div class="flex-row-center buttons-group xsmall-gap">
          <TextEditorStyleToolbar
            textEditor={editor}
            textFormatCategories={[
              TextFormatCategory.Heading,
              TextFormatCategory.TextDecoration,
              TextFormatCategory.Link,
              TextFormatCategory.List,
              TextFormatCategory.Quote,
              TextFormatCategory.Code,
              TextFormatCategory.Table
            ]}
            formatButtonSize={buttonSize}
            {textNodeActions}
            on:focus={() => {
              needFocus = true
            }}
            on:action={(event) => {
              currentTextNodeAction = textNodeActions.find((action) => action.id === event.detail)
              needFocus = true
            }}
          />
        </div>
        <div class="flex-grow" />
        {#if comparedVersion !== undefined}
          <div class="flex-row-center buttons-group xsmall-gap">
            <StyleButton
              icon={IconObjects}
              size={buttonSize}
              selected={showDiff}
              showTooltip={{ label: textEditorPlugin.string.EnableDiffMode }}
              on:click={() => {
                showDiff = !showDiff
                editor.chain().focus()
              }}
            />
            <slot name="tools" />
          </div>
        {:else}
          <div class="formatPanelRef formatPanel buttons-group xsmall-gap">
            <slot name="tools" />
          </div>
        {/if}
      </div>
    {:else if comparedVersion !== undefined}
      <div class="formatPanelRef formatPanel flex flex-grow flex-reverse">
        <StyleButton
          icon={IconObjects}
          size={buttonSize}
          selected={showDiff}
          showTooltip={{ label: textEditorPlugin.string.EnableDiffMode }}
          on:click={() => {
            showDiff = !showDiff
            editor.chain().focus()
          }}
        />
        <slot name="tools" />
      </div>
    {/if}
    <div class="textInput" class:focusable>
      <div class="select-text" style="width: 100%;" bind:this={element} />
    </div>
  </div>
{/if}

<style lang="scss" global>
  .ProseMirror {
    flex-grow: 1;
    min-height: inherit !important;
    max-height: inherit !important;
    outline: none;
    line-height: 150%;
    color: var(--accent-color);

    p:not(:last-child) {
      margin-block-end: 1em;
    }

    pre {
      white-space: pre !important;
    }

    > * + * {
      margin-top: 0.75em;
    }

    /* Placeholder (at the top) */
    p.is-editor-empty:first-child::before {
      content: attr(data-placeholder);
      float: left;
      color: var(--dark-color);
      pointer-events: none;
      height: 0;
    }

    &::-webkit-scrollbar-thumb {
      background-color: var(--scrollbar-bar-color);
    }
    &::-webkit-scrollbar-thumb:hover {
      background-color: var(--scrollbar-bar-hover);
    }
    &::-webkit-scrollbar-corner {
      background-color: var(--scrollbar-bar-color);
    }
    &::-webkit-scrollbar-track {
      margin: 0;
    }
  }
  /* Placeholder (at the top) */
  .ProseMirror p.is-editor-empty:first-child::before {
    color: #adb5bd;
    content: attr(data-placeholder);
    float: left;
    height: 0;
    pointer-events: none;
  }

  .lint-icon {
    display: inline-block;
    position: absolute;
    right: 2px;
    cursor: pointer;
    border-radius: 100px;
    // background: #f22;
    color: white;
    font-family: times, georgia, serif;
    font-size: 15px;
    font-weight: bold;
    width: 0.7em;
    height: 0.7em;
    text-align: center;
    padding-left: 0.5px;
    line-height: 1.1em;
    &.add {
      background: lightblue;
    }
    &.delete {
      background: orange;
    }
  }

  /* Give a remote user a caret */
  .collaboration-cursor__caret {
    border-left: 1px solid #0d0d0d;
    border-right: 1px solid #0d0d0d;
    margin-left: -1px;
    margin-right: -1px;
    pointer-events: none;
    position: relative;
    word-break: normal;
  }

  /* Render the username above the caret */
  .collaboration-cursor__label {
    border-radius: 3px 3px 3px 0;
    color: #0d0d0d;
    font-size: 12px;
    font-style: normal;
    font-weight: 600;
    left: -1px;
    line-height: normal;
    padding: 0.1rem 0.3rem;
    position: absolute;
    top: -1.4em;
    user-select: none;
    white-space: nowrap;
  }

  cmark {
    border-top: 1px solid lightblue;
    border-bottom: 1px solid lightblue;
    border-radius: 2px;
  }

  span.insertion {
    border-top: 1px solid lightblue;
    border-bottom: 1px solid lightblue;
    border-radius: 2px;
  }
  span.deletion {
    text-decoration: line-through;
  }
  .autoOverflow {
    overflow: auto;
  }

  .ref-container .formatPanel {
    margin: -0.5rem -0.25rem 0.5rem;
    padding: 0.375rem;
    background-color: var(--theme-comp-header-color);
    border-radius: 0.5rem;
    box-shadow: var(--button-shadow);
    z-index: 1;
  }

  .ref-container:focus-within .formatPanel {
    position: sticky;
    top: 1.25rem;
  }

  .actionPanel {
    margin: -0.5rem -0.25rem 0.5rem;
    padding: 0.375rem;
    background-color: var(--theme-comp-header-color);
    border-radius: 0.5rem;
    box-shadow: var(--theme-popup-shadow);
    z-index: 1;
  }
</style>
