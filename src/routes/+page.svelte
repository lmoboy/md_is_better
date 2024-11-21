<script>
    import { marked } from 'marked';
    import { Sun, Moon, ChevronDown, ChevronRight } from 'lucide-svelte';
    
    let darkMode = false;
    let selectedColor = '#ffeb3b'; // Default highlight color
    let showColorPicker = false;
    let recentColors = []; 
    let defaultColors = [
        '#ffeb3b', // Yellow
        '#4caf50', // Green
        '#2196f3', // Blue
        '#f44336', // Red
        '#9c27b0'  // Purple
    ];
    let uploadError = '';
    let headers = [];
    let parsedContent;
    let searchQuery = '';
    let filteredHeaders = [];
    let projectTitle = '';
    let collapsedHeaders = new Set();

    // Example markdown content - replace with your actual content
    let markdownContent = `# Welcome to MD is Better
Start by uploading your markdown file using the upload button in the toolbar.

## Quick Start
1. Click the upload button
2. Select a .md file
3. Your content will appear here
`;

    // Load recent colors from localStorage
    if (typeof window !== 'undefined') {
        const savedColors = localStorage.getItem('recentColors');
        recentColors = savedColors ? JSON.parse(savedColors) : defaultColors;
        const lastUsedColor = localStorage.getItem('lastUsedColor');
        if (lastUsedColor) {
            selectedColor = lastUsedColor;
        }
    }

    function updateRecentColors(color) {
        // Remove the color if it already exists
        recentColors = recentColors.filter(c => c !== color);
        // Add the new color to the start
        recentColors = [color, ...recentColors.slice(0, 4)];
        // Save to localStorage
        if (typeof window !== 'undefined') {
            localStorage.setItem('recentColors', JSON.stringify(recentColors));
            localStorage.setItem('lastUsedColor', color);
        }
    }

    function selectColor(color) {
        selectedColor = color;
        updateRecentColors(color);
    }

    // Watch for color changes
    $: if (selectedColor) {
        if (typeof window !== 'undefined') {
            localStorage.setItem('lastUsedColor', selectedColor);
        }
    }

    function parseHeaders() {
        const lines = markdownContent.split('\n');
        headers = [];
        let currentHeader = null;
        
        // First pass: find h1 headers
        const h1Headers = lines.filter(line => line.startsWith('# '));
        
        // If there's exactly one h1 header, use it as project title and exclude from navigation
        if (h1Headers.length === 1) {
            projectTitle = h1Headers[0].replace('# ', '').trim();
        } else {
            projectTitle = ''; // Reset if no single h1
        }

        // Second pass: process all headers
        for (let line of lines) {
            if (line.startsWith('#')) {
                const level = line.match(/^#+/)[0].length;
                const title = line.replace(/^#+\s+/, '').trim();
                
                // Skip if this is the project title (single h1)
                if (projectTitle && level === 1 && title === projectTitle) {
                    continue;
                }

                const header = {
                    title,
                    level,
                    subheaders: []
                };

                if (level === 1 || level === 2) {
                    headers.push(header);
                    currentHeader = header;
                } else if (currentHeader) {
                    currentHeader.subheaders.push(header);
                }
            }
        }
    }

    // Parse headers whenever markdown content changes
    $: {
        parseHeaders();
        parsedContent = marked(markdownContent);
    }

    // Update filtered headers whenever headers or search query changes
    $: {
        if (searchQuery.trim() === '') {
            filteredHeaders = headers;
        } else {
            const query = searchQuery.toLowerCase();
            filteredHeaders = headers.map(header => {
                // Filter subheaders
                const matchingSubheaders = header.subheaders.filter(subheader =>
                    subheader.title.toLowerCase().includes(query)
                );
                
                // If main header matches or has matching subheaders, include it
                if (header.title.toLowerCase().includes(query) || matchingSubheaders.length > 0) {
                    return {
                        ...header,
                        subheaders: matchingSubheaders
                    };
                }
                return null;
            }).filter(Boolean); // Remove null entries
        }
    }

    async function handleFileUpload(event) {
        const file = event.target.files[0];
        if (!file) return;

        // Check if it's a markdown file
        if (!file.name.toLowerCase().endsWith('.md')) {
            uploadError = 'Please upload a markdown (.md) file';
            setTimeout(() => uploadError = '', 3000);
            return;
        }

        try {
            const text = await file.text();
            markdownContent = text;
            uploadError = '';
            // Reset file input
            event.target.value = '';
        } catch (error) {
            uploadError = 'Error reading file. Please try again.';
            setTimeout(() => uploadError = '', 3000);
        }
    }

    function scrollToHeader(title, level) {
        const escapedTitle = title.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        // Remove any existing highlights
        document.querySelectorAll('.header-highlight').forEach(el => {
            el.classList.remove('header-highlight');
        });
        
        // Find and scroll to the header
        const headers = document.querySelectorAll(`h${level}`);
        for (const header of headers) {
            if (header.textContent.trim() === title.trim()) {
                header.scrollIntoView({ behavior: 'smooth', block: 'start' });
                header.classList.add('header-highlight');
                // Remove the highlight after animation
                setTimeout(() => {
                    header.classList.remove('header-highlight');
                }, 2000);
                break;
            }
        }
    }

    function toggleDarkMode() {
        darkMode = !darkMode;
        if (typeof window !== 'undefined') {
            localStorage.setItem('darkMode', darkMode);
        }
    }

    function isInsideHighlight(node) {
        let current = node;
        while (current && current.nodeType === Node.ELEMENT_NODE) {
            if (current.classList?.contains('highlight-text')) {
                return true;
            }
            current = current.parentElement;
        }
        return false;
    }

    function highlightSelection() {
        const selection = window.getSelection();
        if (!selection || selection.isCollapsed) return;
        
        const range = selection.getRangeAt(0);
        const selectedText = range.toString();
        
        // Create a document fragment with a single highlight span
        const fragment = document.createDocumentFragment();
        const span = document.createElement('span');
        span.style.backgroundColor = selectedColor;
        span.className = 'highlight-text';
        span.textContent = selectedText;
        fragment.appendChild(span);

        // Replace the selection with our highlighted fragment
        range.deleteContents();
        range.insertNode(fragment);
        
        // Clear the selection
        selection.removeAllRanges();
    }

    function removeHighlight(event) {
        if (event.target.classList.contains('highlight-text')) {
            const parent = event.target.parentNode;
            const text = event.target.textContent;
            parent.replaceChild(document.createTextNode(text), event.target);
        }
    }

    // Add keyboard shortcut handler
    function handleKeyboardShortcut(event) {
        if ((event.ctrlKey || event.metaKey) && event.key === 'h') {
            event.preventDefault();
            const selection = window.getSelection();
            if (!selection.isCollapsed) {
                const range = selection.getRangeAt(0);
                const selectedText = range.toString();
                
                // Check if the selection is already highlighted
                const parentSpan = range.commonAncestorContainer.parentElement;
                if (parentSpan && parentSpan.classList.contains('highlight')) {
                    return;
                }

                const span = document.createElement('span');
                span.className = 'highlight';
                span.style.backgroundColor = selectedColor;
                
                range.surroundContents(span);
                selection.removeAllRanges();
            }
        }
    }

    function toggleHeader(headerTitle) {
        if (collapsedHeaders.has(headerTitle)) {
            collapsedHeaders.delete(headerTitle);
        } else {
            collapsedHeaders.add(headerTitle);
        }
        collapsedHeaders = collapsedHeaders; // Trigger reactivity
    }

    if (typeof window !== 'undefined') {
        darkMode = localStorage.getItem('darkMode') === 'true';
        window.addEventListener('keydown', handleKeyboardShortcut);
    }
</script>

<div class={`flex h-screen ${darkMode ? 'bg-gray-900' : 'bg-gray-50'}`}
    on:keydown={handleKeyboardShortcut}
>
    <!-- Sidebar -->
    <div class={`w-64 ${darkMode ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'} border-r overflow-y-auto`}>
        <div class="p-4">
            <div class="flex justify-between items-center mb-4">
                <h2 class={`text-lg font-semibold ${darkMode ? 'text-gray-200' : 'text-gray-700'}`}>
                    M.I.B <p style="font-size: 0.8em; font-weight: normal">(MD Is Better)</p>
                </h2>
                <button
                    on:click={toggleDarkMode}
                    class={`p-2 rounded-lg ${darkMode ? 'bg-gray-700 text-yellow-400 hover:bg-gray-600' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'}`}
                >
                    {#if darkMode}
                        <Sun size={18} />
                    {:else}
                        <Moon size={18} />
                    {/if}
                </button>
            </div>

            <!-- Search Bar -->
            <div class="mb-4">
                <input
                    type="text"
                    bind:value={searchQuery}
                    placeholder="Search headers..."
                    class={`w-full px-3 py-2 rounded-lg border ${
                        darkMode 
                            ? 'bg-gray-700 border-gray-600 text-gray-200 placeholder-gray-400' 
                            : 'bg-white border-gray-300 text-gray-700 placeholder-gray-500'
                    } focus:outline-none focus:ring-2 ${
                        darkMode
                            ? 'focus:ring-blue-500 focus:border-blue-500'
                            : 'focus:ring-blue-400 focus:border-blue-400'
                    }`}
                />
            </div>

            <!-- Project Title (if exists) -->
            {#if projectTitle}
                <div class={`py-3 mb-4 ${darkMode ? 'border-gray-700' : 'border-gray-200'} border-b`}>
                    <h3 class={`text-lg font-semibold ${darkMode ? 'text-gray-200' : 'text-gray-700'}`}>
                        {projectTitle}
                    </h3>
                </div>
            {/if}

            <!-- Headers List -->
            {#if filteredHeaders.length === 0}
                <p class={`text-sm ${darkMode ? 'text-gray-400' : 'text-gray-500'} text-center py-2`}>
                    No matching headers found
                </p>
            {/if}
            {#each filteredHeaders as header}
                <div>
                    <button 
                        class={`w-full text-left flex items-center justify-between ${darkMode ? 'text-gray-300 hover:bg-gray-700' : 'text-gray-700 hover:bg-gray-100'} px-2 py-1 rounded cursor-pointer font-medium group`}
                        on:click={() => header.subheaders.length ? toggleHeader(header.title) : scrollToHeader(header.title, header.level)}
                    >
                        <span class="flex-1">{header.title}</span>
                        {#if header.subheaders.length > 0}
                            <span class={`transform transition-transform ${!collapsedHeaders.has(header.title) ? 'rotate-0' : '-rotate-90'}`}>
                                <ChevronDown size={16} />
                            </span>
                        {/if}
                    </button>
                    {#if header.subheaders.length > 0 && !collapsedHeaders.has(header.title)}
                        <div class="ml-4 space-y-1 mt-1">
                            {#each header.subheaders as subheader}
                                <button 
                                    class={`w-full text-left flex items-center ${darkMode ? 'text-gray-400 hover:bg-gray-700' : 'text-gray-600 hover:bg-gray-100'} px-2 py-1 rounded cursor-pointer text-sm`}
                                    on:click={() => scrollToHeader(subheader.title, subheader.level)}
                                >
                                    {subheader.title}
                                </button>
                            {/each}
                        </div>
                    {/if}
                </div>
            {/each}
        </div>
    </div>

    <!-- Main content -->
    <div class="flex-1 flex flex-col overflow-hidden">
        <!-- Toolbar -->
        <div class={`p-2 ${darkMode ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'} border-b flex items-center gap-2 flex-wrap`}>
            <div class="flex items-center gap-2">
                <div class="relative">
                    <button
                        class="w-8 h-8 rounded cursor-pointer border-2 border-gray-300 flex items-center justify-center"
                        style="background-color: {selectedColor}"
                        on:click={() => showColorPicker = !showColorPicker}
                        title="Choose highlight color"
                    >
                        {#if showColorPicker}
                            <span class="transform rotate-180">▼</span>
                        {:else}
                            <span>▼</span>
                        {/if}
                    </button>
                    
                    {#if showColorPicker}
                        <div 
                            class="absolute top-full left-0 mt-1 p-2 rounded shadow-lg {darkMode ? 'bg-gray-800' : 'bg-white'} border {darkMode ? 'border-gray-700' : 'border-gray-200'}"
                            style="min-width: 200px;"
                        >
                            <div class="mb-2">
                                <label class="block text-sm mb-1 {darkMode ? 'text-gray-300' : 'text-gray-600'}">
                                    Custom Color
                                </label>
                                <input
                                    type="color"
                                    bind:value={selectedColor}
                                    on:input={() => {
                                        updateRecentColors(selectedColor);
                                    }}
                                    class="w-full h-8 rounded cursor-pointer"
                                />
                            </div>
                            <div>
                                <label class="block text-sm mb-1 {darkMode ? 'text-gray-300' : 'text-gray-600'}">
                                    Recent Colors
                                </label>
                                <div class="grid grid-cols-5 gap-1">
                                    {#each recentColors as color}
                                        <button
                                            class="w-8 h-8 rounded cursor-pointer border-2 transition-transform hover:scale-110 {color === selectedColor ? 'border-blue-500' : darkMode ? 'border-gray-600' : 'border-gray-300'}"
                                            style="background-color: {color}"
                                            on:click={() => {
                                                selectColor(color);
                                                showColorPicker = false;
                                            }}
                                            title={color}
                                        />
                                    {/each}
                                </div>
                            </div>
                        </div>
                    {/if}
                </div>
                <button
                    on:click={highlightSelection}
                    class={`px-3 py-1 rounded ${darkMode ? 'bg-gray-600 text-gray-200 hover:bg-gray-500' : 'bg-white text-gray-700 hover:bg-gray-50'} flex items-center gap-2`}
                    title="Highlight selected text (Ctrl+H)"
                >
                    <span>Highlight</span>
                    <span class="text-xs opacity-75">(Ctrl+H)</span>
                </button>
            </div>
            <label class="flex items-center gap-2">
                <input
                    type="file"
                    accept=".md"
                    on:change={handleFileUpload}
                    class="hidden"
                />
                <span class={`px-3 py-1 rounded cursor-pointer ${darkMode ? 'bg-gray-700 text-gray-200 hover:bg-gray-600' : 'bg-gray-100 text-gray-700 hover:bg-gray-200'}`}>
                    Upload MD
                </span>
            </label>
            {#if uploadError}
                <div class="text-red-500 text-sm">{uploadError}</div>
            {/if}
        </div>
        <!-- Main content area -->
        <main class={`flex-1 overflow-y-auto ${darkMode ? 'bg-gray-800' : 'bg-white'} p-6`}>
            <div 
                class={`w-full h-full p-4 ${
                    darkMode 
                        ? 'text-gray-200 prose-invert' 
                        : 'text-gray-700'
                } prose max-w-none`}
                on:dblclick={removeHighlight}
            >
                {@html parsedContent}
            </div>
        </main>
    </div>
</div>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
    }

    :global(.prose pre) {
        background-color: rgb(45 55 72 / var(--tw-bg-opacity));
        padding: 1rem;
        border-radius: 0.375rem;
    }

    :global(.highlight-text) {
        cursor: pointer;
        padding: 0 2px;
        transition: background-color 0.3s ease;
    }

    :global(.highlight-text:hover::after) {
        content: '(double-click to remove)';
        position: absolute;
        font-size: 12px;
        background: rgba(0, 0, 0, 0.8);
        color: white;
        padding: 4px 8px;
        border-radius: 4px;
        margin-top: -25px;
        margin-left: 10px;
    }

    :global(.header-highlight) {
        position: relative;
        animation: glow 0.5s ease-in-out;
    }

    @keyframes glow {
        0% {
            box-shadow: 0 0 0 0 rgba(59, 130, 246, 0.5);
        }
        70% {
            box-shadow: 0 0 0 10px rgba(59, 130, 246, 0);
        }
        100% {
            box-shadow: 0 0 0 0 rgba(59, 130, 246, 0);
        }
    }

    :global(.header-highlight::after) {
        content: '';
        position: absolute;
        top: 0;
        left: -10px;
        right: -10px;
        bottom: 0;
        background: linear-gradient(
            90deg,
            transparent 0%,
            var(--highlight-color, rgba(255, 235, 59, 0.2)) 20%,
            var(--highlight-color, rgba(255, 235, 59, 0.2)) 80%,
            transparent 100%
        );
        animation: highlight 2s ease-out forwards;
        pointer-events: none;
        z-index: -1;
        border-radius: 4px;
    }

    @keyframes highlight {
        0% {
            opacity: 1;
        }
        100% {
            opacity: 0;
        }
    }

    :global(.dark .header-highlight::after) {
        --highlight-color: rgba(255, 235, 59, 0.15);
    }
</style>
