<template>
  <div class="opacity-0 show-if-initialized" :class="{ initialized }">
    <pre ref="editor" :class="styles"></pre>
  </div>
</template>

<script>
import ace from "ace-builds"
import "ace-builds/webpack-resolver"
import "ace-builds/src-noconflict/ext-language_tools"
import "ace-builds/src-noconflict/mode-graphqlschema"
import * as gql from "graphql"
import { getAutocompleteSuggestions } from "graphql-language-service-interface"
import { defineComponent } from "@nuxtjs/composition-api"
import { defineGQLLanguageMode } from "~/helpers/syntax/gqlQueryLangMode"
import debounce from "~/helpers/utils/debounce"

export default defineComponent({
  props: {
    value: {
      type: String,
      default: "",
    },
    theme: {
      type: String,
      required: false,
      default: null,
    },
    onRunGQLQuery: {
      type: Function,
      default: () => {},
    },
    options: {
      type: Object,
      default: () => {},
    },
    styles: {
      type: String,
      default: "",
    },
  },

  data() {
    return {
      initialized: false,
      editor: null,
      cacheValue: "",
      validationSchema: null,
    }
  },

  computed: {
    appFontSize() {
      return getComputedStyle(document.documentElement).getPropertyValue(
        "--body-font-size"
      )
    },
  },

  watch: {
    value(value) {
      if (value !== this.cacheValue) {
        this.editor.session.setValue(value, 1)
        this.cacheValue = value
      }
    },
    theme() {
      this.initialized = false
      this.editor.setTheme(`ace/theme/${this.defineTheme()}`, () => {
        this.$nextTick().then(() => {
          this.initialized = true
        })
      })
    },
    options(value) {
      this.editor.setOptions(value)
    },
  },

  mounted() {
    defineGQLLanguageMode(ace)

    const langTools = ace.require("ace/ext/language_tools")

    const editor = ace.edit(this.$refs.editor, {
      mode: `ace/mode/gql-query`,
      enableBasicAutocompletion: true,
      enableLiveAutocompletion: true,
      ...this.options,
    })

    // Set the theme and show the editor only after it's been set to prevent FOUC.
    editor.setTheme(`ace/theme/${this.defineTheme()}`, () => {
      this.$nextTick().then(() => {
        this.initialized = true
      })
    })

    // Set the theme and show the editor only after it's been set to prevent FOUC.
    editor.setTheme(`ace/theme/${this.defineTheme()}`, () => {
      this.$nextTick().then(() => {
        this.initialized = true
      })
    })

    editor.setFontSize(this.appFontSize)

    const completer = {
      getCompletions: (
        editor,
        _session,
        { row, column },
        _prefix,
        callback
      ) => {
        if (this.validationSchema) {
          const completions = getAutocompleteSuggestions(
            this.validationSchema,
            editor.getValue(),
            {
              line: row,
              character: column,
            }
          )

          callback(
            null,
            completions.map(({ label, detail }) => ({
              name: label,
              value: label,
              score: 1.0,
              meta: detail,
            }))
          )
        } else {
          callback(null, [])
        }
      },
    }

    langTools.setCompleters([completer])

    if (this.value) editor.setValue(this.value, 1)

    this.editor = editor
    this.cacheValue = this.value

    editor.commands.addCommand({
      name: "runGQLQuery",
      exec: () => this.onRunGQLQuery(this.editor.getValue()),
      bindKey: {
        mac: "cmd-enter",
        win: "ctrl-enter",
      },
    })

    editor.commands.addCommand({
      name: "prettifyGQLQuery",
      exec: () => this.prettifyQuery(),
      bindKey: {
        mac: "cmd-p",
        win: "ctrl-p",
      },
    })

    editor.on("change", () => {
      const content = editor.getValue()
      this.$emit("input", content)
      this.parseContents(content)
      this.cacheValue = content
    })

    this.parseContents(this.value)
  },

  beforeDestroy() {
    this.editor.destroy()
  },

  methods: {
    prettifyQuery() {
      try {
        this.$emit("update-query", gql.print(gql.parse(this.editor.getValue())))
      } catch (e) {
        this.$toast.error(this.$t("error.gql_prettify_invalid_query"), {
          icon: "error_outline",
        })
      }
    },

    defineTheme() {
      if (this.theme) {
        return this.theme
      }
      const strip = (str) =>
        str.replace(/#/g, "").replace(/ /g, "").replace(/"/g, "")
      return strip(
        window
          .getComputedStyle(document.documentElement)
          .getPropertyValue("--editor-theme")
      )
    },

    setValidationSchema(schema) {
      this.validationSchema = schema
      this.parseContents(this.cacheValue)
    },

    parseContents: debounce(function (content) {
      if (content !== "") {
        try {
          const doc = gql.parse(content)

          if (this.validationSchema) {
            this.editor.session.setAnnotations(
              gql
                .validate(this.validationSchema, doc)
                .map(({ locations, message }) => ({
                  row: locations[0].line - 1,
                  column: locations[0].column - 1,
                  text: message,
                  type: "error",
                }))
            )
          }
        } catch (e) {
          this.editor.session.setAnnotations([
            {
              row: e.locations[0].line - 1,
              column: e.locations[0].column - 1,
              text: e.message,
              type: "error",
            },
          ])
        }
      } else {
        this.editor.session.setAnnotations([])
      }
    }, 2000),
  },
})
</script>

<style scoped lang="scss">
.show-if-initialized {
  &.initialized {
    @apply opacity-100;
  }

  & > * {
    @apply transition-none;
  }
}
</style>
