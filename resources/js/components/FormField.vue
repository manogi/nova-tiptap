<template>
    <default-field
        :field="field"
        :errors="errors"
        :full-width-content="true"
        :show-help-text="showHelpText"
    >
        <template slot="field">
            <editor-menu-bar :editor="editor">
                <div class="menubar" slot-scope="{ commands, isActive, getMarkAttrs }">
                    <div class="toolbar">
                        <template v-for="(buttonKey, params) in buttons">
                            <template v-if="buttonKey == 'heading' || params == 'heading'">
                                <heading-buttons
                                    :key="'button-'+buttonKey"
                                    :headingLevels="headingLevels"
                                    :commands="commands"
                                    :isActive="isActive"
                                >
                                </heading-buttons>
                            </template>

                            <template v-if="buttonKey == 'table' || params == 'table'">
                                <table-buttons
                                    :key="'button-'+buttonKey"
                                    :commands="commands"
                                    :isActive="isActive"
                                >
                                </table-buttons>
                            </template>

                            <template v-if="
                                buttonKey != 'heading'
                                && buttonKey != 'link'
                                && params != 'heading'
                                && buttonKey != 'table'
                            ">
                                <normal-button
                                    :key="'button-'+buttonKey"
                                    :buttonKey="buttonKey"
                                    :commands="customCommands(commands)"
                                    :isActive="isActive"
                                >
                                </normal-button>
                            </template>

                            <span
                                :key="'button-'+buttonKey"
                                class="tiptap-button-container"
                                v-if="buttonKey == 'link'"
                            >
                                <link-button
                                    :commands="commands"
                                    :isActive="isActive"
                                    :linkMenuIsActive="linkMenuIsActive"
                                    :linkUrl="linkUrl"
                                    :hideLinkMenu="hideLinkMenu"
                                    :showLinkMenu="showLinkMenu"
                                    :getMarkAttrs="getMarkAttrs"
                                    :setLinkUrl="setLinkUrl"
                                >
                                </link-button>
                            </span>

                        </template>
                    </div>
                </div>
            </editor-menu-bar>

            <editor-content            
                :id="field.attribute"
                class="
                tiptap-content
                py-3 h-auto
                pr-6
                pb-4
                pt-4
                w-full
                form-control
                form-input
                form-input-bordered
                mt-2
                no-focus
                "
                :editor="editor"
                v-if="!editHTML"
            />
            <textarea
                    v-model="value"
                    @change="updateEditorValue"
                    v-if="editHTML"
                    class="w-full form-control form-input form-input-bordered py-3 h-auto edit-html"
            ></textarea>
        </template>
    </default-field>
</template>

<script>

import { FormField, HandlesValidationErrors } from 'laravel-nova';
import { Editor, EditorContent, EditorMenuBar, EditorMenuBubble } from 'tiptap';
import HeadingButtons from './buttons/HeadingButtons';
import TableButtons from './buttons/TableButtons';
import NormalButton from './buttons/NormalButton';
import LinkButton from './buttons/LinkButton';

import {
    Blockquote,
    CodeBlock,
    HardBreak,
    Heading,
    OrderedList,
    BulletList,
    HorizontalRule,
    ListItem,
    TodoItem,
    TodoList,
    Bold,
    Code,
    Italic,
    Link,
    Table,
    TableHeader,
    TableCell,
    TableRow,
    Strike,
    Underline,
    History,
} from 'tiptap-extensions';

import { Superscript } from 'tiptap-extension-superscript';
import { Subscript } from 'tiptap-extension-subscript';

import Iframe from '../extensions/iframe.js';

export default {
    mixins: [FormField, HandlesValidationErrors],

    props: ['resourceName', 'resourceId', 'field'],

    components: {
        EditorContent,
        EditorMenuBar,
        HeadingButtons,
        TableButtons,
        NormalButton,
        LinkButton,
    },

    data: function () {
        return {
            headingLevels: 6,

            linkUrl: null,

            linkMenuIsActive: false,

            editor: null,

            editHTML: false,
        }
    },

    computed: {
        buttons() {
            return this.field.buttons ? this.field.buttons : ['bold', 'italic'];
        }
    },

    methods: {
        initEditor() {
            let outsideScope = this;

            this.editor = new Editor({
                onUpdate(state){
                    outsideScope.value = state.getHTML();
                },
                extensions: [
                    new Blockquote(),
                    new BulletList(),
                    new CodeBlock(),
                    new HardBreak(),
                    new Heading({ levels: [1, 2, 3, 4, 5, 6] }),
                    new HorizontalRule(),
                    new ListItem(),
                    new OrderedList(),
                    new TodoItem(),
                    new TodoList(),
                    new Bold(),
                    new Code(),
                    new Italic(),
                    new Link({
                        openOnClick: false,
                    }),
                    new Strike(),
                    new Underline(),
                    new History(),
                    new Table({
                        resizable: true,
                    }),
                    new TableHeader(),
                    new TableCell(),
                    new TableRow(),
                    new Superscript(),
                    new Subscript(),
                    new Iframe(),
                ],
                editorProps: {
                    handleKeyDown: (editorView, keyboardEvent) => {
                        // Prevent ? or / from triggering Nova global search
                        keyboardEvent.stopPropagation();
                    }
                }
            });

            this.editor.setContent(this.value);

            // set heading levels
            if (this.field.headingLevels) {
                this.headingLevels = this.field.headingLevels;
            } else {
                // fallback for the old style like this: 'heading' => 4
                _.forEach(this.buttons, function(params, key) {
                    if (key == 'heading') {
                        this.headingLevels = params;
                    }
                }.bind(this));
            }

        },

        showLinkMenu(attrs) {
            this.linkUrl = attrs.href;
            this.linkMenuIsActive = true;
        },

        hideLinkMenu() {
            this.linkUrl = null;
            this.linkMenuIsActive = false;
        },

        setLinkUrl(command, url) {
            command({ href: url });
            this.hideLinkMenu();
            this.editor.focus();
        },

        updateEditorValue() {
            this.editor.setContent(this.value);
        },

        customCommands(commands) {

            commands.edit_html = function () {
                this.editHTML = !this.editHTML;
                if (this.editHTML) {
                    this.value = pretty(this.value);
                }
            }.bind(this);

            return commands;
        }
    },

    mounted: function () {
        this.initEditor();
    }
}
</script>

<style>
.tiptap-content {
    outline: none !important;
    box-shadow: none !important;
}

.tiptap-content .ProseMirror:focus {
    outline: none;
}

.tiptap-content .ProseMirror.resize-cursor {
    cursor: col-resize;
}

.tiptap-content .ProseMirror ins {
    text-decoration: none;
    background-color: highlight;
}

.tiptap-content .ProseMirror p, .tiptap-content .ProseMirror ul, .tiptap-content .ProseMirror ol, .tiptap-content .ProseMirror blockquote, .tiptap-content .ProseMirror pre {
    margin-bottom: 1em;
}

.tiptap-content .ProseMirror h1, .tiptap-content .ProseMirror h2, .tiptap-content .ProseMirror h3, .tiptap-content .ProseMirror h4, .tiptap-content .ProseMirror h5, .tiptap-content .ProseMirror h6 {
    margin-bottom: 0.5em;
}

.tiptap-content .relationship-tabs-panel .ProseMirror h1 {
    display: block;
}

.tiptap-content .ProseMirror pre {
    white-space: pre-line;
}

.tiptap-content .ProseMirror blockquote {
    border-left: 4px solid var(--60);
    padding-left: 12px;
    margin-left: 0;
    font-style: italic;
}

.tiptap-content .ProseMirror p:last-child {
    margin-bottom: 0;
}

.tiptap-content .ProseMirror td p {
    margin-bottom: 0;
}
.tiptap-content .ProseMirror td p:last-child {
    margin-bottom: 0;
}

.tiptap-content .ProseMirror table {
    border: 1px solid var(--60);
}

.tiptap-content .ProseMirror td.selectedCell {
    background-color: blanchedalmond;
}

.tiptap-content .ProseMirror td {
    border: 1px solid var(--60);
    padding: 6px;
}

.tiptap-button {
    box-sizing: border-box;
    vertical-align: top;
    margin-right: 8px;
    font-weight: 700;
    margin-bottom: 8px;
}
.tiptap-button.is-thin {
    font-weight: 400;
}

.tiptap-button-container {
    position: relative;
    overflow: visible;
}

.tiptap-button-form {
    position: absolute;
    top: 36px;
    left: -130px;
}

.tiptap-button-form .form-input {
    width: 300px;
    margin-right: 20px;
    color: black;
}

.tiptap-button-form .btn {
    width: 16px;
    height: 16px;
}


.tiptap-button::before {
    margin: 0;
    height: 16px;
    line-height: 0;
    position: relative;
}

.tiptap-button.is-bold::before {
    content: 'B';
}

.tiptap-button.is-italic::before {
    font-style: italic;
    content: 'I';
}

.tiptap-button.is-strike::before {
    text-decoration: line-through;
    content: 'S';
}

.tiptap-button.is-underline::before {
    text-decoration: underline;
    content: 'U';
}

.tiptap-button.is-code::before {
    content: '<>';
}

.tiptap-button.is-code_block::before {
    content: '</>';
}

.tiptap-content .edit-html {
    min-height:400px;
    margin-top:10px;
}

.tiptap-content .iframe__embed{
    border: 0;
}
.tiptap-content .iframe__input {
    display: block;
    width: 100%;
    font: inherit;
    border: 0;
    border-radius: 5px;
    background-color: rgba(0,0,0, 0.1);
    padding: 0.3rem 0.5rem;
    margin-bottom: 0.5rem;
}
.tiptap-content hr {
    display: block;
    border-top: 1px solid var(--70);
    margin-top: 0.5rem;
    margin-bottom: 0.5rem;
}
</style>
