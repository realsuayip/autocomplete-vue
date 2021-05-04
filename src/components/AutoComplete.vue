<template>
  <div>
    <input type="text"
           autocomplete="off"
           role="combobox"
           aria-autocomplete="list"
           :aria-owns="'lb-' + uid"
           :aria-expanded="!!suggestions.length ? 'true' : 'false'"
           :aria-activedescendant="activeDescendant"
           :id="uid"
           v-model="query"
           ref="inputElement"
           @input="input"
           @keydown="keydown"
           @focusout="focusout">
    <ul v-if="suggestions.length" role="listbox" :id="'lb-' + uid" ref="ulElement">
      <li v-for="(item, index) in suggestions"
          :role="item._isNoResultsMsg ? 'alert' : 'option'"
          :aria-live="item._isNoResultsMsg ? 'assertive' : false"
          :key="item.value"
          :id="uid + '-cb-opt-' + index"
          :class="item.selected ? 'selected' : ''"
          @mouseover="mouseover(index)"
          @click="_select(index)"
          v-html="highlight && !item._isNoResultsMsg ? _highlight(item.name) : _escape(item.name)">
      </li>
    </ul>
  </div>
</template>

<script>
import {popperGenerator, defaultModifiers} from "@popperjs/core/lib/popper-lite"
import flip from "@popperjs/core/lib/modifiers/flip"
import preventOverflow from "@popperjs/core/lib/modifiers/preventOverflow"

let instance = 0

export default {
  name: "AutoComplete",
  props: {
    highlight: Boolean,
    cache: Boolean,
    noResultsMessage: String,
    inputId: String
  },

  data () {
    return {
      emptyResults: [],
      cachedResults: [],
      suggestions: [],
      selected: null,
      popperInstance: null,
      query: null,
      inputQuery: null,  // To make highlight feature correctly.
      activeDescendant: null,
      completed: false,
    }
  },

  created () {
    this.uid = this.inputId || `i-autocomplete-${instance}`
    instance += 1
  },

  updated () {
    // Destroy popper instance if no results are showing.
    if (!this.suggestions.length) {
      this.popperInstance && this.popperInstance.destroy()
      this.popperInstance = null
      return
    }

    // Create popper instance if not exists, update otherwise.
    if (!this.popperInstance) {
      const createPopper = popperGenerator({
        defaultModifiers: [...defaultModifiers, flip, preventOverflow],
      })
      this.popperInstance = createPopper(this.$refs.inputElement, this.$refs.ulElement)
    } else {
      this.popperInstance.update()
    }

    // Set activeDescendant if any suggestion is selected.
    let selectedIndex = this.suggestions.findIndex(i => i.selected)
    this.activeDescendant = selectedIndex !== -1 && `${this.uid}-cb-opt-${selectedIndex}`
  },

  methods: {
    // -- Methods for highlighting feature. --
    _escape (value) {
      const entityMap = { "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }
      return value.replace(/[&<>"']/g, s => entityMap[s])
    },
    _escapeRegex: value => value.replace(/[|\\{}()[\]^$+*?.]/g, "\\$&"),
    _highlight (suggestion) {
      /**
       *  highlight function source:
       *  Ajax Autocomplete for jQuery, version 1.4.11
       *  (c) 2017 Tomas Kirda
       *  https://github.com/devbridge/jQuery-Autocomplete
       *  MIT
       */
      const query = this.inputQuery || this.query

      if (!query) {
        return this._escape(suggestion)
      }

      const pattern = "(" + this._escapeRegex(query) + ")"

      return this._escape(
          suggestion.replace(new RegExp(pattern, "gi"), "<mark>$1</mark>")
      ).replace(/&lt;(\/?mark)&gt;/g, "<$1>")
    },

    // -- Internal methods --
    _unload (completed) {
      this.completed = completed
      this.suggestions = []
      this.activeDescendant = null
    },

    _notFound () {
      if (this.noResultsMessage === undefined) {
        return
      }
      this.suggestions = [{ name: this.noResultsMessage, value: null, _isNoResultsMsg: true }]
    },

    _select (index, unload = true) {
      const selected = this.suggestions[index]

      if (selected._isNoResultsMsg) {
        return
      }

      if (unload) {
        this._unload(true)
        this.$emit('selected', selected)
      }

      this.query = selected.name
    },

    _setCache (query, data) {
      if (!this.cache) {
        return
      }

      if (!data.length) {
        this.emptyResults.push(query)
        return
      }

      this.cachedResults[query] = data
    },

    _getCache (query) {
      if (!this.cache) {
        return false
      }

      return (this.emptyResults.some(key => query.startsWith(key)) && []) || this.cachedResults[query]
    },

    suggest () {
      const currentQuery = this.query.trim()
      const cached = this._getCache(currentQuery)

      if (cached) {
        if (cached.length) {
          this.suggestions = cached
        } else {
          this._notFound()
        }
        return
      }

      this.$emit('lookup', currentQuery, (query, data) => {
        if (!data.length) {
          this._setCache(query, [])
          return this._notFound()
        }

        const items = data.map(item => {
          item.selected = false
          return item
        })

        this._setCache(query, items)

        if (query === this.query.trim()) {
          // Ignores outdated callback results. If you are making requests
          // you probably want to cancel them in the callback.
          this.suggestions = items
        }
      })
    },

    // -- Events --
    input () {
      const query = this.query.trim()
      this.inputQuery = this.query
      this.selected = null

      if (query) {
        return this.suggest()
      }

      this._unload()
    },

    keydown (event) {
      if (!this.suggestions.length || !["ArrowUp", "ArrowDown", "Escape", "Enter", "Tab"].includes(event.key)) {
        return
      }

      const selectedIndex = this.suggestions.findIndex(i => i.selected)

      switch (event.key) {
        case "ArrowDown": {
          if (selectedIndex === -1) {
            this.suggestions[0].selected = true
            return
          }

          this.suggestions[selectedIndex].selected = false

          if (selectedIndex === this.suggestions.length - 1) {
            this.suggestions[0].selected = true
            return
          }

          this.suggestions[selectedIndex + 1].selected = true
          break
        }

        case "ArrowUp": {
          if (this.suggestions.findIndex(i => i._isNoResultsMsg) === -1) {
            // Prevents cursor from moving to beginning of the input.
            event.preventDefault()
          }

          if (selectedIndex === -1) {
            this.suggestions[this.suggestions.length - 1].selected = true
            return
          }

          this.suggestions[selectedIndex].selected = false

          if (selectedIndex === 0) {
            this.suggestions[this.suggestions.length - 1].selected = true
            return
          }

          this.suggestions[selectedIndex - 1].selected = true
          break
        }

        case "Enter": {
          event.preventDefault() // Prevents form submit
          this._select(selectedIndex)
          break
        }

        case "Escape": {
          this._unload()
          break
        }

        case "Tab": {
          const selected = this.suggestions[selectedIndex]
          if (selected && !event.shiftKey) {
            this.query = selected.name
          }
          break
        }
      }

      if (["ArrowUp", "ArrowDown"].includes(event.key)) {
        this._select(this.suggestions.findIndex(i => i.selected), false)
      }

    },

    mouseover (index) {
      this.suggestions.forEach((item, idx) => {
        item.selected = idx === index
      })
    },

    focusout () {
      setTimeout(() => {
        this._unload(true)
      }, 200)
    }
  }
}
</script>

<style scoped>

input {
  width: 100%;
}

div {
  position: relative;
}

ul {
  margin: 0;
  width: 100%;
  border: 1px solid #999;
  border-radius: 3px;
  background: #fff;
  padding-left: 0;
  list-style: none;
  overflow: auto;
  z-index: 1;
}

ul > li {
  padding: .3em 1em;
  white-space: nowrap;
  cursor: default;
}

ul > li.selected {
  background: #f8f9fa;
}

ul > li[role=alert] {
  text-align: center;
}

ul >>> mark {
  color: inherit;
  background: none;
  font-weight: 700;
}
</style>
