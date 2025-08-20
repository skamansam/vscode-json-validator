<h1>Welcome to your library project</h1>
<p>Create your package using @sveltejs/package and preview/showcase your work with SvelteKit</p>
<p>Visit <a href="https://svelte.dev/docs/kit">svelte.dev/docs/kit</a> to read the documentation</p>

<script lang="ts">
  import { onDestroy, onMount } from 'svelte';
  import { parse, type ParseError, getParseErrorMessage } from '$lib/json-validator';
  import { EditorView, lineNumbers } from '@codemirror/view';
  import { EditorState } from '@codemirror/state';
  import { json } from '@codemirror/lang-json';
  import { linter, type Diagnostic } from '@codemirror/lint';

  type UIError = {
    message: string;
    offset: number;
    length: number;
    line: number;
    column: number;
    endLine: number;
    endColumn: number;
  };

  function computeLineStarts(text: string): number[] {
    const starts = [0];
    for (let i = 0; i < text.length; i++) {
      const ch = text.charCodeAt(i);
      if (ch === 10 /*\n*/) {
        starts.push(i + 1);
      } else if (ch === 13 /*\r*/) {
        if (i + 1 < text.length && text.charCodeAt(i + 1) === 10 /*\n*/) i++;
        starts.push(i + 1);
      }
    }
    return starts;
  }

  function offsetToPosition(offset: number, starts: number[]) {
    if (offset <= 0) return { line: 1, column: 1 };
    let low = 0, high = starts.length - 1, mid = 0;
    while (low <= high) {
      mid = (low + high) >> 1;
      const start = starts[mid];
      const next = mid + 1 < starts.length ? starts[mid + 1] : Number.MAX_SAFE_INTEGER;
      if (offset < start) high = mid - 1;
      else if (offset >= next) low = mid + 1;
      else return { line: mid + 1, column: offset - start + 1 };
    }
    const last = starts[starts.length - 1] ?? 0;
    return { line: starts.length, column: Math.max(1, offset - last + 1) };
  }

  function humanizeErrors(errs: ParseError[], text: string): UIError[] {
    const starts = computeLineStarts(text);
    return errs.map((e) => {
      const start = offsetToPosition(e.offset, starts);
      const end = offsetToPosition(Math.min(e.offset + Math.max(e.length, 1), text.length), starts);
      return {
        message: getParseErrorMessage(e.error) || 'JSON parse error',
        offset: e.offset,
        length: e.length,
        line: start.line,
        column: start.column,
        endLine: end.line,
        endColumn: end.column
      } satisfies UIError;
    });
  }

  let input = '{\n  "hello": "world"\n}';
  let errors: UIError[] = [];

  let editorParent: HTMLDivElement;
  let editor: EditorView | undefined;

  function runValidate() {
    const errs: ParseError[] = [];
    // parse fills errs; result is unused here
    parse(input, errs, { allowTrailingComma: true });
    errors = humanizeErrors(errs, input);
  }

  const jsonLinter = linter((view) => {
    const text = view.state.doc.toString();
    const errs: ParseError[] = [];
    parse(text, errs, { allowTrailingComma: true });
    const diags: Diagnostic[] = errs.map((e) => ({
      from: e.offset,
      to: Math.min(text.length, e.offset + Math.max(e.length, 1)),
      severity: 'error',
      message: getParseErrorMessage(e.error)
    }));
    return diags;
  });

  onMount(() => {
    runValidate();
    editor = new EditorView({
      state: EditorState.create({
        doc: input,
        extensions: [
          lineNumbers(),
          json(),
          jsonLinter,
          EditorView.updateListener.of((update) => {
            if (update.docChanged) {
              input = update.state.doc.toString();
              runValidate();
            }
          })
        ]
      }),
      parent: editorParent
    });
  });

  onDestroy(() => editor?.destroy());
</script>

<section class="prose max-w-none mb-6">
  <h2>Plain textarea validator</h2>
  <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
    <div>
      <label class="block text-sm font-medium mb-2">JSON input</label>
      <textarea
        class="w-full h-64 rounded border p-3 font-mono"
        bind:value={input}
        on:input={runValidate}
        spellcheck="false"
        aria-label="JSON input"
      />
    </div>
    <div>
      <label class="block text-sm font-medium mb-2">Validation errors</label>
      {#if errors.length === 0}
        <div class="text-green-700 bg-green-50 border border-green-200 rounded p-3">
          No errors. JSON is valid.
        </div>
      {:else}
        <ul class="space-y-2">
          {#each errors as e}
            <li class="bg-red-50 border border-red-200 rounded p-3">
              <div class="font-medium text-red-800">{e.message}</div>
              <div class="text-sm text-red-700">Line {e.line}, Col {e.column}</div>
            </li>
          {/each}
        </ul>
      {/if}
    </div>
  </div>
  <div class="mt-2">
    <button class="px-3 py-1.5 bg-blue-600 text-white rounded" on:click={runValidate}>
      Validate
    </button>
  </div>
  <hr class="my-6" />
  <h2>CodeMirror editor with inline errors</h2>
  <div bind:this={editorParent} class="border rounded min-h-[16rem]" aria-label="CodeMirror JSON editor"></div>
  <p class="text-sm text-gray-600 mt-2">Diagnostics are shown inline via CodeMirror's linter.</p>
  <p class="text-xs text-gray-500">Try adding a trailing comma or breaking a string to see errors.</p>
</section>
