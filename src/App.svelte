<script>
  import { onMount } from 'svelte';
  import { openDB } from 'idb';

  let pages = [];
  let currentPageIndex = 0;
  let title = '';
  let note = '';

  let db;

  onMount(async () => {
    db = await openDB('notesDB', 1, {
      upgrade(db) {
        db.createObjectStore('notes', { keyPath: 'title' });
        db.createObjectStore('pages', { keyPath: 'index' });
      }
    });
    await loadPages();
  });

  async function loadPages() {
    const tx = db.transaction('pages', 'readonly');
    const store = tx.objectStore('pages');
    const allPages = await store.getAll();

    if (allPages.length > 0) {
      pages = allPages.sort((a, b) => a.index - b.index).map(page => page.title);
      title = pages[currentPageIndex];
      note = (await db.get('notes', title)).note;
    } else {
      addPage();
    }
  }

  async function saveNote() {
    const storedPageName = pages[currentPageIndex];
    if (storedPageName !== title) {
      await db.delete('notes', storedPageName);
      pages[currentPageIndex] = title;
      await savePages();
    }
    await db.put('notes', { title, note });
  }

  async function savePages() {
    const tx = db.transaction('pages', 'readwrite');
    const store = tx.objectStore('pages');
    await store.clear();
    pages.forEach((page, index) => {
      store.put({ index, title: page });
    });
  }

  async function addPage() {
    pages.push("New Page");
    await savePages();
    selectPage(pages.length ? pages.length - 1 : 0);
  }

  async function selectPage(index) {
    currentPageIndex = index;
    title = pages[currentPageIndex];
    const noteData = await db.get('notes', title);
    note = noteData ? noteData.note : '';
  }

  async function deletePage(index) {
    const pageTitle = pages[index];
    pages.splice(index, 1);
    await db.delete('notes', pageTitle);
    if (index === currentPageIndex) {
      currentPageIndex = Math.max(0, index - 1);
      if (pages.length > 0) {
        selectPage(currentPageIndex);
      } else {
        addPage();
      }
    } else if (index < currentPageIndex) {
      currentPageIndex--;
    }
    await savePages();
  }
</script>

<aside class="fixed top-0 left-0 z-40 w-60 h-screen">
  <div class="bg-light-gray overflow-y-auto py-5 px-3 h-full border-r border-gray-200">
    <ul class="space-y-2">
      {#each pages as page, index}
        <li class="flex justify-between items-center">
          <button on:click={() => selectPage(index)} class="{index == currentPageIndex ? 'bg-dark-gray' : ''} py-2 px-3 text-gray-900 rounded-lg flex-1 text-left">
            {page}
          </button>
          <button on:click={() => deletePage(index)} class="text-red-500 ml-2">🗑️</button>
        </li>
      {/each}
      <li class="text-center">
        <button on:click={addPage} class="font-medium">+ Add page</button>
      </li>
    </ul>
  </div>
</aside>

<main class="p-4 ml-60 h-auto">
  <div class="grid grid-cols-2 items-center mb-3">
    <h1 class="text-3xl font-bold" contenteditable bind:textContent={title}></h1>
    <button class="ml-auto bg-gray-800 text-white px-5 py-2.5 rounded-lg font-medium text-sm mt-3 hover:bg-gray-900" on:click={saveNote}>Save</button>
  </div>
  <hr/>
  <textarea class="mt-3 block w-full bg-gray-50 border border-gray-300 rounded-lg text-gray-900 p-2.5" bind:value={note}></textarea>
</main>

<style>
  .bg-light-gray {
    background: #FBFBFB;
  }
  .bg-dark-gray {
    background: #EFEFEF;
  }
</style>
