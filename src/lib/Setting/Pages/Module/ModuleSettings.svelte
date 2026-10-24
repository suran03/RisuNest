<script lang="ts">
    import { language } from "src/lang";
    import { DBState } from 'src/ts/stores.svelte';
    import Button from "src/lib/UI/GUI/Button.svelte";
    import TextInput from "src/lib/UI/GUI/TextInput.svelte";
    import ModuleMenu from "src/lib/Setting/Pages/Module/ModuleMenu.svelte";
    import { exportModule, exportModuleLegacy, importModule, type RisuModule } from "src/ts/process/modules";
    import {
        SquarePen, TrashIcon, Globe, Share2Icon, PlusIcon, HardDriveUpload,
        Waypoints, UserIcon, FolderIcon, FolderPlusIcon, ChevronDownIcon,
        ChevronRightIcon, GripVerticalIcon, ArrowUpIcon, ArrowDownIcon
    } from "@lucide/svelte";
    import { v4 } from "uuid";
    import { tooltip } from "src/ts/gui/tooltip";
    import { alertConfirm, alertError, alertInput, alertNormal, alertSelect } from "src/ts/alert";
    import { importMCPModule } from "src/ts/process/mcp/mcp";
    import { convertModuleToCharacter } from "src/ts/interchangeability";
    import { commitDetachedCharacter } from "src/ts/characters";
    import { commitModuleCharacterConversion } from "./moduleCharacterConversion";
    import { collectOwnedAssetKeys, scheduleDeletedOwnerAssetCleanup } from "src/ts/storage/deletedOwnerAssetCleanup";
    import { assetAliasCleanupTarget } from "src/ts/storage/persistentDataRuntime.svelte";
    import Sortable from "sortablejs";
    import { sortableOptions } from "src/ts/util";

    export interface ModuleFolder {
        id: string;
        name: string;
    }

    type FolderAwareModule = RisuModule & { folderId?: string };

    const FOLDERS_STORAGE_KEY = "risunest-module-folders-v1";
    const COLLAPSED_STORAGE_KEY = "risunest-module-folders-collapsed-v1";

    let mode = $state(0);
    let editModuleId = $state('');
    const currentModule = $derived(DBState.db.modules.find(module => module.id === editModuleId));
    let moduleSearch = $state('');
    let charConversionMode = $state(false);

    // Load folders from DBState (if present) or localStorage fallback
    function loadInitialFolders(): ModuleFolder[] {
        const dbAny = DBState.db as Record<string, unknown>;
        if (Array.isArray(dbAny.moduleFolders) && dbAny.moduleFolders.length > 0) {
            return dbAny.moduleFolders as ModuleFolder[];
        }
        try {
            const raw = localStorage.getItem(FOLDERS_STORAGE_KEY);
            if (raw) return JSON.parse(raw) as ModuleFolder[];
        } catch {}
        return [];
    }

    function loadInitialCollapsed(): Set<string> {
        try {
            const raw = localStorage.getItem(COLLAPSED_STORAGE_KEY);
            if (raw) return new Set(JSON.parse(raw) as string[]);
        } catch {}
        return new Set();
    }

    let folders = $state<ModuleFolder[]>(loadInitialFolders());
    let collapsed = $state<Set<string>>(loadInitialCollapsed());
    let rootEl: HTMLDivElement | undefined = $state();

    function saveFolders(next: ModuleFolder[]) {
        folders = next;
        try {
            (DBState.db as Record<string, unknown>).moduleFolders = next;
        } catch {}
        try {
            localStorage.setItem(FOLDERS_STORAGE_KEY, JSON.stringify(next));
        } catch {}
    }

    function toggleCollapsed(folderId: string) {
        const next = new Set(collapsed);
        if (next.has(folderId)) next.delete(folderId);
        else next.add(folderId);
        collapsed = next;
        try {
            localStorage.setItem(COLLAPSED_STORAGE_KEY, JSON.stringify([...next]));
        } catch {}
    }

    $effect(() => {
        if (mode !== 0 && !currentModule) mode = 0;
    });

    const query = $derived(moduleSearch.trim().toLowerCase());
    const dragDisabled = $derived(query.length > 0);

    function matchesSearch(rmodule: RisuModule): boolean {
        if (!query) return true;
        const nameMatch = rmodule.name?.toLowerCase().includes(query);
        const descMatch = rmodule.description?.toLowerCase().includes(query);
        return !!(nameMatch || descMatch);
    }

    interface FolderGroup {
        folder: ModuleFolder | null; // null = Uncategorized
        items: { module: FolderAwareModule; index: number }[];
    }

    const groups = $derived.by<FolderGroup[]>(() => {
        const byId = new Map<string, { module: FolderAwareModule; index: number }[]>();
        for (const f of folders) byId.set(f.id, []);
        const uncategorized: { module: FolderAwareModule; index: number }[] = [];

        (DBState.db.modules as FolderAwareModule[]).forEach((m, index) => {
            const bucket = m.folderId ? byId.get(m.folderId) : undefined;
            if (bucket) bucket.push({ module: m, index });
            else uncategorized.push({ module: m, index });
        });

        return [
            ...folders.map(folder => ({ folder, items: byId.get(folder.id) ?? [] })),
            { folder: null, items: uncategorized }
        ];
    });

    function isGlobal(rmodule: RisuModule) {
        return DBState.db.enabledModules.includes(rmodule.id);
    }

    function isIntegrated(rmodule: RisuModule) {
        return !!rmodule.namespace &&
            !!DBState.db.moduleIntergration?.split(',').map((s) => s.trim()).includes(rmodule.namespace);
    }

    function toggleGlobal(rmodule: RisuModule) {
        if (isGlobal(rmodule)) {
            DBState.db.enabledModules.splice(DBState.db.enabledModules.indexOf(rmodule.id), 1);
        } else {
            DBState.db.enabledModules.push(rmodule.id);
        }
        DBState.db.enabledModules = DBState.db.enabledModules;
        charConversionMode = false;
    }

    async function createFolder() {
        const name = (await alertInput((language as Record<string, string>).folderNameInput ?? "폴더 이름을 입력하세요"))?.trim();
        if (!name) return;
        saveFolders([...folders, { id: v4(), name }]);
    }

    async function renameFolder(folder: ModuleFolder) {
        const name = (await alertInput((language as Record<string, string>).folderNameInput ?? "새 폴더 이름을 입력하세요", [], folder.name))?.trim();
        if (!name || name === folder.name) return;
        saveFolders(folders.map(f => f.id === folder.id ? { ...f, name } : f));
    }

    async function deleteFolder(folder: ModuleFolder) {
        const confirmMsg = (language as Record<string, string>).folderDeleteKeepItems ?? `"${folder.name}" 폴더를 삭제할까요? (내부 모듈은 미분류로 이동됩니다)`;
        if (!(await alertConfirm(confirmMsg))) return;
        const modules = DBState.db.modules as FolderAwareModule[];
        for (const m of modules) {
            if (m.folderId === folder.id) {
                delete m.folderId;
            }
        }
        DBState.db.modules = [...modules];
        saveFolders(folders.filter(f => f.id !== folder.id));
    }

    function moveFolder(folder: ModuleFolder, delta: -1 | 1) {
        const from = folders.findIndex(f => f.id === folder.id);
        const to = from + delta;
        if (from < 0 || to < 0 || to >= folders.length) return;
        const next = [...folders];
        next.splice(from, 1);
        next.splice(to, 0, folder);
        saveFolders(next);
    }

    async function moveModuleToFolder(rmodule: FolderAwareModule) {
        const uncategorizedLabel = (language as Record<string, string>).folderUncategorized ?? "미분류 (Uncategorized)";
        const cancelLabel = language.cancel ?? "취소";
        const options = [...folders.map(f => `📁 ${f.name}`), `📂 ${uncategorizedLabel}`, cancelLabel];
        const sel = parseInt(await alertSelect(options));
        if (Number.isNaN(sel) || sel < 0 || sel >= options.length - 1) return;
        const targetFolderId = sel < folders.length ? folders[sel].id : undefined;

        const modules = DBState.db.modules as FolderAwareModule[];
        const idx = modules.findIndex(m => m.id === rmodule.id);
        if (idx < 0) return;
        const updated = { ...modules[idx], folderId: targetFolderId };
        if (!targetFolderId) delete updated.folderId;
        modules[idx] = updated;
        DBState.db.modules = [...modules];
    }

    function applyDomPlacements() {
        if (!rootEl) return;
        const modules = DBState.db.modules as FolderAwareModule[];
        const byId = new Map(modules.map(m => [m.id, m]));
        const next: FolderAwareModule[] = [];

        rootEl.querySelectorAll<HTMLElement>('[data-folder-container]').forEach(container => {
            const folderId = container.dataset.folderContainer || undefined;
            container.querySelectorAll<HTMLElement>('[data-module-id]').forEach(el => {
                const id = el.dataset.moduleId;
                if (!id) return;
                const mod = byId.get(id);
                if (mod) {
                    const copy: FolderAwareModule = { ...mod, folderId };
                    if (!folderId) delete copy.folderId;
                    next.push(copy);
                    byId.delete(id);
                }
            });
        });

        // Preserve any remaining modules if any
        for (const leftover of byId.values()) {
            next.push(leftover);
        }

        if (next.length === modules.length) {
            DBState.db.modules = next;
        }
    }

    function sortableFolderContainer(node: HTMLElement) {
        let instance: Sortable | null = null;
        $effect(() => {
            instance?.destroy();
            if (dragDisabled) return;
            instance = Sortable.create(node, {
                ...sortableOptions,
                animation: 150,
                group: 'risunest-module-folders',
                draggable: '[data-folder-id]',
                handle: '.folder-drag-handle',
                ghostClass: 'opacity-40',
                onEnd: () => {
                    const orderedIds = Array.from(node.querySelectorAll<HTMLElement>(':scope > [data-folder-id]'))
                        .map(el => el.dataset.folderId)
                        .filter((id): id is string => !!id);
                    const byId = new Map(folders.map(f => [f.id, f]));
                    const next = orderedIds.map(id => byId.get(id)).filter((f): f is ModuleFolder => !!f);
                    if (next.length === folders.length) {
                        saveFolders(next);
                    }
                }
            });
            return () => {
                instance?.destroy();
                instance = null;
            };
        });
    }

    function sortableItemContainer(node: HTMLElement) {
        let instance: Sortable | null = null;
        $effect(() => {
            instance?.destroy();
            if (dragDisabled) return;
            instance = Sortable.create(node, {
                ...sortableOptions,
                animation: 150,
                group: 'risunest-module-items',
                draggable: '[data-module-id]',
                filter: '.no-sort',
                preventOnFilter: false,
                ghostClass: 'opacity-40',
                onEnd: () => {
                    applyDomPlacements();
                }
            });
            return () => {
                instance?.destroy();
                instance = null;
            };
        });
    }
</script>

{#if mode === 0}
    <div class="flex items-center justify-between mt-2 mb-2">
        <h2 class="text-2xl font-bold">{language.modules}</h2>
        <div class="flex items-center gap-1.5">
            <button
                class="flex items-center gap-1 px-2.5 py-1.5 text-sm rounded-md border border-selected bg-darkbg text-textcolor hover:border-blue-500 cursor-pointer transition-colors"
                onclick={createFolder}
                use:tooltip={"새 폴더 만들기"}
            >
                <FolderPlusIcon size={16} />
                <span>{(language as Record<string, string>).folderNew ?? "새 폴더"}</span>
            </button>
        </div>
    </div>

    <!-- 상단 액션 툴바 & 검색창 -->
    <div class="flex flex-wrap items-center gap-2 mt-2">
        <button
            class="flex items-center gap-1 px-2.5 py-1.5 text-sm rounded-md bg-selected text-textcolor hover:bg-blue-600 hover:text-white cursor-pointer transition-colors"
            onclick={() => {
                const tempModule = { name: '', description: '', id: v4() };
                DBState.db.modules.push(tempModule);
                editModuleId = tempModule.id;
                mode = 1;
            }}
        >
            <PlusIcon size={16} />
            <span>{language.createModule}</span>
        </button>

        <button
            class="flex items-center gap-1 px-2.5 py-1.5 text-sm rounded-md border border-selected bg-darkbg text-textcolor2 hover:text-textcolor cursor-pointer transition-colors"
            onclick={() => importModule()}
            use:tooltip={language.importModule}
        >
            <HardDriveUpload size={16} />
            <span>{language.importModule}</span>
        </button>

        <button
            class="flex items-center gap-1 px-2.5 py-1.5 text-sm rounded-md border border-selected bg-darkbg text-textcolor2 hover:text-textcolor cursor-pointer transition-colors"
            onclick={() => importMCPModule()}
            use:tooltip={"Import MCP Module"}
        >
            <Waypoints size={16} />
            <span>MCP</span>
        </button>

        <button
            class="flex items-center gap-1 px-2.5 py-1.5 text-sm rounded-md border border-selected {charConversionMode ? 'bg-violet-600 text-white border-violet-500' : 'bg-darkbg text-textcolor2 hover:text-textcolor'} cursor-pointer transition-colors"
            onclick={() => { charConversionMode = !charConversionMode; }}
            use:tooltip={language.convertToCharacter ?? "캐릭터로 변환 모드"}
        >
            <UserIcon size={16} />
        </button>
    </div>

    <TextInput className="mt-3" placeholder={language.search} bind:value={moduleSearch} />

    <!-- 폴더 및 모듈 리스트 영역 -->
    <div class="contain w-full max-w-full mt-3 flex flex-col gap-2.5 flex-1 overflow-y-auto pb-4" bind:this={rootEl}>
        {#if DBState.db.modules.length === 0 && folders.length === 0}
            <div class="text-textcolor2 p-4 border border-selected rounded-md text-center">{language.noModules}</div>
        {:else}
            <!-- 사용자 정의 폴더 목록 (드래그 앤 드롭 정렬 지원) -->
            <div class="flex flex-col gap-2.5" use:sortableFolderContainer>
                {#each groups as group (group.folder?.id ?? '__uncategorized__')}
                    {#if group.folder}
                        {@const folder = group.folder}
                        {@const isCollapsed = collapsed.has(folder.id)}
                        <div data-folder-id={folder.id} class="rounded-md border border-selected bg-darkbg overflow-hidden">
                            <!-- 폴더 헤더 -->
                            <div
                                class="flex items-center gap-2 px-3 py-2.5 bg-selected/40 text-textcolor cursor-pointer select-none"
                                role="button"
                                tabindex="0"
                                onclick={() => toggleCollapsed(folder.id)}
                                onkeydown={(e) => { if (e.key === 'Enter') toggleCollapsed(folder.id); }}
                            >
                                <span class="folder-drag-handle text-textcolor2 hover:text-textcolor cursor-grab active:cursor-grabbing p-0.5" onclick={(e) => e.stopPropagation()}>
                                    <GripVerticalIcon size={16} />
                                </span>
                                {#if isCollapsed}
                                    <ChevronRightIcon size={16} class="shrink-0 text-textcolor2" />
                                {:else}
                                    <ChevronDownIcon size={16} class="shrink-0 text-textcolor2" />
                                {/if}
                                <FolderIcon size={16} class="shrink-0 text-blue-400" />
                                <span class="font-medium truncate grow">{folder.name}</span>
                                <span class="text-xs px-2 py-0.5 rounded-full bg-darkbg text-textcolor2">{group.items.length}</span>

                                <!-- 폴더 관리 버튼들 -->
                                <div class="no-sort flex items-center gap-1 ml-1" onclick={(e) => e.stopPropagation()}>
                                    <button class="p-1 text-textcolor2 hover:text-textcolor cursor-pointer" use:tooltip={"위로 이동"} onclick={() => moveFolder(folder, -1)}>
                                        <ArrowUpIcon size={14} />
                                    </button>
                                    <button class="p-1 text-textcolor2 hover:text-textcolor cursor-pointer" use:tooltip={"아래로 이동"} onclick={() => moveFolder(folder, 1)}>
                                        <ArrowDownIcon size={14} />
                                    </button>
                                    <button class="p-1 text-textcolor2 hover:text-green-400 cursor-pointer" use:tooltip={"폴더 이름 변경"} onclick={() => renameFolder(folder)}>
                                        <SquarePen size={14} />
                                    </button>
                                    <button class="p-1 text-textcolor2 hover:text-red-400 cursor-pointer" use:tooltip={"폴더 삭제"} onclick={() => deleteFolder(folder)}>
                                        <TrashIcon size={14} />
                                    </button>
                                </div>
                            </div>

                            <!-- 폴더 내부 아이템 컨테이너 -->
                            <div
                                data-folder-container={folder.id}
                                class:hidden={isCollapsed}
                                class="flex flex-col p-2 gap-1 min-h-10"
                                use:sortableItemContainer
                            >
                                {#each group.items as item (item.module.id)}
                                    {@render moduleRow(item.module)}
                                {:else}
                                    <div class="no-sort text-xs text-textcolor2 text-center py-2">폴더가 비어 있습니다 (여기로 모듈을 드래그하세요)</div>
                                {/each}
                            </div>
                        </div>
                    {/if}
                {/each}
            </div>

            <!-- 미분류 (Uncategorized) 폴더 영역 -->
            {#each groups as group (group.folder?.id ?? '__uncategorized__')}
                {#if !group.folder}
                    {@const isCollapsed = collapsed.has('')}
                    <div class="rounded-md border border-selected bg-darkbg overflow-hidden">
                        <div
                            class="flex items-center gap-2 px-3 py-2.5 bg-selected/20 text-textcolor2 cursor-pointer select-none"
                            role="button"
                            tabindex="0"
                            onclick={() => toggleCollapsed('')}
                            onkeydown={(e) => { if (e.key === 'Enter') toggleCollapsed(''); }}
                        >
                            {#if isCollapsed}
                                <ChevronRightIcon size={16} class="shrink-0" />
                            {:else}
                                <ChevronDownIcon size={16} class="shrink-0" />
                            {/if}
                            <FolderIcon size={16} class="shrink-0" />
                            <span class="font-medium truncate grow">{(language as Record<string, string>).folderUncategorized ?? "미분류 (Uncategorized)"}</span>
                            <span class="text-xs px-2 py-0.5 rounded-full bg-darkbg text-textcolor2">{group.items.length}</span>
                        </div>

                        <div
                            data-folder-container=""
                            class:hidden={isCollapsed}
                            class="flex flex-col p-2 gap-1 min-h-10"
                            use:sortableItemContainer
                        >
                            {#each group.items as item (item.module.id)}
                                {@render moduleRow(item.module)}
                            {:else}
                                <div class="no-sort text-xs text-textcolor2 text-center py-2">{language.noModules}</div>
                            {/each}
                        </div>
                    </div>
                {/if}
            {/each}
        {/if}
    </div>

{:else if mode === 1 && currentModule}
    <h2 class="mb-2 text-2xl font-bold mt-2">{language.createModule}</h2>
    <ModuleMenu {currentModule}/>
    <Button className="mt-6" onclick={() => {
        mode = 0;
    }}>{language.createModule}</Button>
{:else if mode === 2 && currentModule}
    <h2 class="mb-2 text-2xl font-bold mt-2">{language.editModule}</h2>
    <ModuleMenu {currentModule}/>
    {#if currentModule.name !== ''}
        <Button className="mt-6" onclick={() => {
            mode = 0;
        }}>{language.editModule}</Button>
    {/if}
{/if}

{#snippet moduleRow(rmodule: FolderAwareModule)}
    <div
        data-module-id={rmodule.id}
        class:hidden={!matchesSearch(rmodule)}
        class="flex items-center gap-2 rounded-md px-2.5 py-2 border border-transparent hover:border-selected hover:bg-selected/30 transition-colors cursor-pointer"
        onclick={() => {
            if (!rmodule.mcp) {
                editModuleId = rmodule.id;
                mode = 2;
            }
        }}
    >
        <!-- 드래그 핸들 -->
        <span class="text-textcolor2/60 hover:text-textcolor cursor-grab active:cursor-grabbing shrink-0">
            <GripVerticalIcon size={16} />
        </span>

        {#if rmodule.mcp}
            <Waypoints size={18} class="shrink-0 text-textcolor2" />
        {/if}

        <!-- 모듈 이름 & 설명 (PocketRisu 스타일 컴팩트 2줄 배치) -->
        <div class="flex flex-col min-w-0 grow">
            <span class="text-textcolor font-medium truncate">{rmodule.name || 'Unnamed Module'}</span>
            <span class="text-xs text-textcolor2 truncate">{rmodule.description || 'No description provided'}</span>
        </div>

        <!-- 우측 액션 아이콘 버튼들 (.no-sort 처리로 드래그 간섭 방지) -->
        <div class="no-sort flex items-center gap-1.5 shrink-0" onclick={(e) => e.stopPropagation()}>
            {#if charConversionMode}
                <button
                    class="p-1.5 rounded hover:bg-selected cursor-pointer text-violet-400"
                    use:tooltip={language.convertToCharacter ?? "캐릭터로 변환"}
                    onclick={async () => {
                        const module = DBState.db.modules.find((v) => v.id === rmodule.id);
                        if (!module) return;
                        const char = convertModuleToCharacter(module);
                        await commitModuleCharacterConversion(char, {
                            commit: commitDetachedCharacter,
                            onSuccess: () => alertNormal(language.successfullyConverted),
                            onError: alertError,
                        });
                    }}
                >
                    <UserIcon size={17} />
                </button>
            {:else}
                <!-- 글로벌 활성화 토글 -->
                <button
                    class="p-1.5 rounded hover:bg-selected cursor-pointer {isGlobal(rmodule) ? 'text-blue-500' : isIntegrated(rmodule) ? 'text-amber-500 hover:text-green-500' : 'text-textcolor2 hover:text-green-500'}"
                    use:tooltip={language.enableGlobal}
                    onclick={() => toggleGlobal(rmodule)}
                >
                    <Globe size={17} />
                </button>

                <!-- 폴더 이동 버튼 (모바일/터치 친화) -->
                <button
                    class="p-1.5 rounded hover:bg-selected text-textcolor2 hover:text-blue-400 cursor-pointer"
                    use:tooltip={(language as Record<string, string>).folderMoveTo ?? "폴더 이동"}
                    onclick={() => moveModuleToFolder(rmodule)}
                >
                    <FolderIcon size={17} />
                </button>

                {#if !rmodule.mcp}
                    <!-- 내보내기(공유) -->
                    <button
                        class="p-1.5 rounded hover:bg-selected text-textcolor2 hover:text-green-500 cursor-pointer"
                        use:tooltip={language.download}
                        onclick={async () => {
                            const sel = parseInt(await alertSelect([`CharX (${language.recommended})`, `RisuM (Legacy)`]));
                            if (sel === 0) exportModule(rmodule);
                            else if (sel === 1) exportModuleLegacy(rmodule);
                        }}
                    >
                        <Share2Icon size={17} />
                    </button>

                    <!-- 수정 -->
                    <button
                        class="p-1.5 rounded hover:bg-selected text-textcolor2 hover:text-green-500 cursor-pointer"
                        use:tooltip={language.edit}
                        onclick={() => {
                            editModuleId = rmodule.id;
                            mode = 2;
                        }}
                    >
                        <SquarePen size={17} />
                    </button>
                {/if}

                <!-- 삭제 (RisuNest 에셋 정리 로직 100% 보존) -->
                <button
                    class="p-1.5 rounded hover:bg-selected text-textcolor2 hover:text-red-400 cursor-pointer"
                    use:tooltip={language.remove}
                    onclick={async () => {
                        const d = await alertConfirm(`${language.removeConfirm}` + rmodule.name);
                        if (d) {
                            const index = DBState.db.modules.findIndex((v) => v.id === rmodule.id);
                            if (index < 0) return;
                            const startedAtMs = Date.now();
                            const assetKeys = collectOwnedAssetKeys(DBState.db.modules[index]);
                            if (DBState.db.enabledModules.includes(rmodule.id)) {
                                DBState.db.enabledModules.splice(DBState.db.enabledModules.indexOf(rmodule.id), 1);
                                DBState.db.enabledModules = DBState.db.enabledModules;
                            }
                            DBState.db.modules.splice(index, 1);
                            DBState.db.modules = DBState.db.modules;
                            scheduleDeletedOwnerAssetCleanup(assetKeys, startedAtMs, 'module-delete', assetAliasCleanupTarget);
                        }
                    }}
                >
                    <TrashIcon size={17} />
                </button>
            {/if}
        </div>
    </div>
{/snippet}
