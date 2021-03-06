<script>
    import { onMount, onDestroy, getContext } from 'svelte';
    import { fade, fly } from 'svelte/transition';
    import CreateCollectionModal from './modals/CreateCollectionModal.svelte';
    import DeleteCollectionModal from './modals/DeleteCollectionModal.svelte';
    import CollectionTile from './tiles/CollectionTile.svelte';
    const { open } = getContext('simple-modal');
    import { deo } from './../stores/dropEventStore.js';
    import EmptyCollectionTile from './tiles/EmptyCollectionTile.svelte';

    // array of BookmarkTreeNode
    let allCollections = [];

    // Called only if pid no longer points to the correct putaway folder.
    function refreshPidAndloadCollections(pid) {
        // store pid in local storage for use later
        chrome.storage.local.set({ "pid": pid });
        loadCollections(pid, false);
    }

    function loadCollections(pid, retry=true){
        chrome.bookmarks.getChildren(pid, function (children) {
            try{
                if(console.log(chrome.runtime.lastError)){
                    throw "Putaway folder pid invalidated, refreshing.";
                }
                // only folders
                allCollections = children.filter((e) => e.url == null);
            }catch(e){
                if(retry){
                    // pid is invalidated, add it again.
                    const putAwayFolderName = 'PutAway';
                    chrome.bookmarks.getTree(function (tree) {
                        var otherBookmarksFolderId = tree[0].children[1].id;
                        chrome.bookmarks.getChildren(otherBookmarksFolderId, function (children) {
                            var putawayfolder = children.find((e) => e.title == putAwayFolderName);
                            var pid;
                            if (!putawayfolder) {
                                // Folder doesn't exist, so we create one
                                chrome.bookmarks.create({
                                    'parentId': otherBookmarksFolderId,
                                    'title': putAwayFolderName,
                                }, function (newFolder) {
                                    pid = newFolder.id;
                                    refreshPidAndloadCollections(pid);
                                });
                            } else {
                                pid = putawayfolder.id;
                                refreshPidAndloadCollections(pid);
                            }

                        });
                    });
                }
            }
        });
    }
    onMount(() => {
        chrome.storage.local.get('pid', function (res) {
            if(res.pid){
                loadCollections(res.pid);
            }else{
                loadCollections('-1');
            }
        });
    });

    var clickAddCollection = async () => {
        var c = await open(CreateCollectionModal);
        if (c) {
            // add to list at its index
            allCollections.splice(c.index, 0, c);
            allCollections = allCollections;
        }
    }

    var onCollectionDrop = (e, dropIndex) => {
        e.preventDefault();
        var rawData = e.dataTransfer.getData('text');

        deo.set({
            source: rawData,
            target: "c" + dropIndex.toString(),
            sourceObj: null,
            targetObj: null
        });
    }

    const unsubsribe = deo.subscribe(obj => {
        if (obj.source[0] == "c" &&
            obj.target[0] == "c") {
            var dragIndex = parseInt(obj.source.substr(1));
            var dropIndex = parseInt(obj.target.substr(1));
            // move allCollections from dragIndex to dropIndex
            if (dragIndex >= dropIndex) {
                chrome.bookmarks.move(allCollections[dragIndex].id, { index: dropIndex });
                allCollections.splice(dropIndex, 0, allCollections[dragIndex]);
                allCollections.splice(dragIndex + 1, 1);
            }
            else {
                chrome.bookmarks.move(allCollections[dragIndex].id, { index: dropIndex });
                allCollections.splice(dropIndex, 0, allCollections[dragIndex]);
                allCollections.splice(dragIndex, 1);
            }
            allCollections = allCollections;
        }
    });
    onDestroy(unsubsribe);

    var clickDeleteCollection = async (index) => {
        var c = await open(DeleteCollectionModal, { collectionName: allCollections[index].title });
        if (c) {
            chrome.bookmarks.removeTree(allCollections[index].id);
            allCollections.splice(index, 1);
            allCollections = allCollections;
        }
    }
</script>
<style>
    .plus-icon-dummy {
        width: 20px;
        height: 20px;
        border-width: 3px;
        border-radius: 100%;
        background: -webkit-linear-gradient(0deg, transparent 0%, transparent 46%, white 46%, white 56%, transparent 56%, transparent 100%), -webkit-linear-gradient(90deg, transparent 0%, transparent 46%, white 46%, white 56%, transparent 56%, transparent 100%);
        border-color: gray;
        background-color: gray;
    }

    .plus-icon {
        position: absolute;
        margin-right: 20px;
        margin-bottom: 8vh;
        bottom: 0;
        right: 0;
        width: 40px;
        height: 40px;
        border-width: 8px;
        border-radius: 100%;
        background: -webkit-linear-gradient(0deg, transparent 0%, transparent 46%, white 46%, white 56%, transparent 56%, transparent 100%), -webkit-linear-gradient(90deg, transparent 0%, transparent 46%, white 46%, white 56%, transparent 56%, transparent 100%);
        border-color: gray;
        background-color: gray;
        z-index: 2;
    }

    .no-collections-indicator {
        position: absolute;
        right: 0;
        top: 0;
        bottom: 0;
        margin-bottom: 100px;
        left: 0;
        display: flex;
        flex-direction: row;
        align-items: center;
        align-content: center;
        justify-content: center;
    }
</style>

<main style="position: relative;">

    {#if allCollections.length==0 }
        <div class="no-collections-indicator">
            <h3 >No Collections, Click '</h3>
            <button class="plus-icon-dummy"></button>
            <h3>' To create one</h3>
        </div>
    {/if}

    <button class="plus-icon pointer" on:click|preventDefault|stopPropagation={clickAddCollection}></button>

    <div class="scroll">
        {#each allCollections as collection,i (collection.id)}
            <CollectionTile {collection} index={i} {onCollectionDrop} {clickDeleteCollection}/>
        {/each}
        <EmptyCollectionTile index={allCollections.length} {onCollectionDrop}/>
    </div>
</main>