<template>
  <div class="slide-wrapper">
    <!-- Navbar -->
    <div class="navbar">
      <h2 class="navbar-title">{{ topic }}</h2>
      <img src="../assets/logo.png" />
    </div>

    <div class="slide-body">
      <!-- Top row: sub-topic badge -->
      <div class="row-top">
        <div class="badge" v-click>{{ subTopic }}</div> <!-- v-click -->
      </div>

      <!-- Main row: editor + sidebar -->
      <div class="row-main">
        <!-- Code editor slot -->
        <div class="col-editor">
          <slot name="editor">
            <JavaRunner v-click/> <!-- v-click -->
          </slot>
        </div>

        <!-- Sidebar: info cards -->
        <div class="col-sidebar">
          <template v-for="(item, index) in contents" :key="index">
            <!-- Code editor card -->
            <div v-if="item.codeEditor" class="code-card" v-click> <!-- v-click -->
              <div class="code-card__lang">{{ item.lang ?? 'text' }}</div>
              <pre class="code-card__pre"><code>{{ item.text }}</code></pre>
            </div>

            <!-- Regular / highlight info card -->
            <div
              v-else
              class="info-card"
              :class="{ 'info-card--highlight': item.highlight }"
              v-html="item.text"
              v-click
            /> <!-- v-click -->

            <br v-if="!item.codeEditor" />
          </template>

          <!-- Optional extra sidebar content -->
          <slot name="sidebar" />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  /** Text shown in the red navbar title bar */
  topic: {
    type: String,
    required: true,
  },
  /** Text shown in the top badge (sub-topic) */
  subTopic: {
    type: String,
    default: '',
  },
  /**
   * Array of content objects. Three variants:
   *
   *   { text }                          → plain info card (HTML supported)
   *   { text, highlight: true }         → green-accented info card
   *   { text, codeEditor: true, lang }  → syntax-highlighted code block
   *
   * `lang` defaults to 'text' (e.g. 'java', 'js', 'python')
   */
  contents: {
    type: Array,
    default: () => [],
  },
});
</script>

<style scoped>
/* ── Layout ────────────────────────────────────────────── */
.slide-wrapper {
  margin-top: -10px;
  margin-left: -30px;
  width: 107%;
  height: 68vh;
  font-size: 0.8rem;
  font-weight: 400;
  overflow-x: auto;
  overflow-y: auto;
  scrollbar-width: none;
}

.slide-body {
  display: flex;
  flex-direction: column;
  border-radius: 4px;
  min-height: 93%;
}

/* ── Navbar ────────────────────────────────────────────── */
.navbar {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  gap: 0.75rem;
  padding: 0 10px;
  color: #ffffff;
}

.navbar > img {
  height: 30px;
}

.navbar-title {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 700;
  background-color: #ef5050;
  color: #ffffff;
  width: 80%;
  padding-left: 10px;
  margin-left: -10px;
  border-radius: 5px;
}

/* ── Content Rows ──────────────────────────────────────── */
.row-top {
  width: 100%;
  height: 7.7vh;
  display: flex;
  flex-direction: row;
  align-items: center;
}

.row-main {
  width: 100%;
  min-height: 55vh;
  display: flex;
  flex-direction: row;
}

/* ── Columns ───────────────────────────────────────────── */
.col-editor {
  width: 65%;
  height: 54vh;
}

.col-sidebar {
  width: 35%;
  height: 54vh;
  padding: 0 10px;
}

/* ── Info Boxes ────────────────────────────────────────── */
.badge {
  border-radius: 4px;
  font-size: 1rem;
  display: inline-block;
  min-width: 32px;
  padding:5px 10px;
  background-color: #ff914d4e;
  border: 1px solid #ff914dd3;
}

.info-card {
  border-radius: 4px;
  font-size: 0.9rem;
  color: #374151;
  background-color: #e2f0fe;
  border: 1px solid #a9c4d2;
  display: inline-block;
  min-width: 32px;
  padding: 5px;
  width: 100%;
  margin-bottom: 5px;
  box-sizing: border-box;
}

.info-card--highlight {
  background-color: #6fcf9745;
  border: 1px solid #1f6f5fa7;
  text-align: center;
}

/* ── Code Card ─────────────────────────────────────────── */
.code-card {
  border-radius: 4px;
  border: 1px solid #334155;
  background-color: #0f172a;
  overflow: hidden;
  width: 100%;
  margin-bottom: 5px;
  box-sizing: border-box;
}

.code-card__lang {
  background-color: #1e293b;
  color: #94a3b8;
  font-size: 0.7rem;
  padding: 2px 8px;
  border-bottom: 1px solid #334155;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.code-card__pre {
  margin: 0;
  padding: 8px 10px;
  overflow-x: auto;
  font-family: 'Fira Code', 'Cascadia Code', monospace;
  font-size: 0.8rem;
  line-height: 1.5;
  color: #e2e8f0;
  white-space: pre;
}
</style>