<template>
  <div class="studio">
    <header class="topBar">
      <router-link class="wordmark" to="/studio">Toonflow</router-link>
      <nav class="mainNav" aria-label="创作导航">
        <router-link v-for="item in navigation" :key="item.name" :to="{ name: item.name }" :class="{ currentNav: item.name === 'studio' && onCreate }" :aria-label="item.label" :title="item.label">
          <component :is="item.icon" :size="20" aria-hidden="true" />
          <span v-if="route.name === item.name || (item.name === 'studio' && onCreate)">{{ item.label }}</span>
        </router-link>
      </nav>
      <router-link class="iconButton settingsLink" :to="{ name: 'studioSettings' }" aria-label="设置" title="设置"><icon-settings :size="21" /></router-link>
    </header>

    <main class="page" :class="{ galleryPage: route.name === 'studio' }">
      <template v-if="showComposer">
        <template v-if="!onCreate">
          <section class="exploreIntro" aria-labelledby="exploreTitle">
            <div>
              <span class="exploreKicker"><span></span> TOONFLOW / EXPLORE</span>
              <h1 id="exploreTitle">灵感图库</h1>
              <p>从一帧画面出发，找到下一次创作的方向。</p>
            </div>
            <router-link class="darkButton" to="/studio/create"><icon-sparkles :size="17" />开始创作</router-link>
          </section>
          <section class="featuredWall" aria-label="精选专题">
            <button v-for="(feature, index) in featured" :key="feature.category" class="featuredCard" type="button" @click="showCollection(feature.category)">
              <img :src="feature.image" alt="" decoding="async" />
              <span class="featureIndex">精选 / 0{{ index + 1 }}</span>
              <span class="featureCaption"><strong>{{ feature.title }}</strong><small>{{ feature.subtitle }}</small></span>
              <span class="featureArrow"><icon-arrow-up-right :size="18" /></span>
            </button>
          </section>
          <section ref="exploreResults" class="exploreResults" aria-labelledby="discoverTitle">
            <div class="discoverHeading"><h2 id="discoverTitle">发现灵感</h2><span>{{ visibleInspirations.length }} 张</span></div>
            <div class="discoverTools">
              <div class="galleryFilters" role="group" aria-label="灵感分类">
                <button v-for="category in categories" :key="category" type="button" :aria-pressed="activeCategory === category" @click="activeCategory = category">{{ category }}</button>
              </div>
              <label class="librarySearch exploreSearch"><icon-search :size="17" /><input v-model.trim="inspirationQuery" type="search" aria-label="搜索灵感" placeholder="搜题材、质感或关键词" /></label>
            </div>
            <div v-if="visibleInspirations.length" class="gallery" aria-label="灵感示例">
              <button v-for="(item, index) in visibleInspirations" :key="item.title" class="galleryCard" :class="item.shape" type="button" @click="openInspiration(item)">
                <img :src="item.url" alt="" :style="{ objectPosition: item.position }" :loading="index < 8 ? 'eager' : 'lazy'" decoding="async" />
                <span class="galleryCaption"><strong>{{ item.title }}</strong><small>{{ item.category }}</small></span>
                <span class="viewInspiration" aria-hidden="true"><icon-arrow-up-right :size="17" /></span>
              </button>
            </div>
            <div v-else class="emptyState"><h2>没有匹配的灵感</h2><button class="pillButton" type="button" @click="resetInspirationFilters">清除筛选</button></div>
          </section>
        </template>
        <section v-else class="creationStage" aria-labelledby="creationTitle">
          <div class="stageHeading">
            <div><span class="stageEyebrow">快速创作</span><h1 id="creationTitle">生成工作台</h1></div>
            <router-link class="pillButton" to="/studio/works">查看生成记录 <icon-arrow-up-right :size="16" /></router-link>
          </div>
          <div class="stagePreview" :class="{ waiting: generating, stageFailed: generationError && !generating }" :aria-busy="generating">
            <template v-if="stageUrl && latest && !generating">
              <img v-if="latest.mediaType === 'image'" :src="stageUrl" alt="生成图片" />
              <video v-else :src="stageUrl" controls playsinline aria-label="生成视频" />
            </template>
            <div v-else class="stageContent">
              <span class="stageOrb"><icon-sparkles :size="28" aria-hidden="true" /></span>
              <h2 role="status">{{ stageTitle }}</h2>
              <p>{{ generationError || stageError || (generating ? "完成后会在这里显示，并保存到「我的作品」。" : latest ? "正在读取生成文件…" : "在下方输入提示词，开始新的创作。") }}</p>
              <button v-if="pendingTask && !generating" class="pillButton" type="button" @click="resume">查询任务</button>
              <button v-else-if="generationError" class="pillButton" type="button" @click="expandComposer">调整提示词</button>
            </div>
          </div>
          <div v-if="!generating && (stagePrompt || stageUrl)" class="stageMeta">
            <p v-if="stagePrompt">{{ stagePrompt }}</p>
            <div v-if="stageUrl && latest" class="stageActions">
              <a class="pillButton" :href="stageUrl" :download="latest.path.split('/').at(-1)">下载</a>
              <button v-if="latest.prompt" class="pillButton" type="button" @click="reuseWork(latest)">再次编辑</button>
              <button v-if="canUseAsReference(latest)" class="darkButton" type="button" @click="referenceWork(latest, 'video')">让它动起来</button>
            </div>
          </div>
        </section>

        <section ref="composer" class="composer" :class="{ collapsed: composerCollapsed, stageComposer: onCreate }" aria-label="快速创作">
          <div class="composerExpand" :inert="composerCollapsed">
            <div class="composerContent">
              <div class="composerHeader">
                <div class="modes" :class="{ videoSelected: mode === 'video' }" role="group" aria-label="创作类型">
                  <button v-for="option in modes" :key="option.id" type="button" :aria-pressed="mode === option.id" @click="changeMode(option.id)">{{ option.label }}</button>
                </div>
                <button class="dockToggle" type="button" aria-label="收起生成框" title="收起生成框" @click="minimizeComposer"><icon-chevron-down :size="19" /></button>
              </div>
              <div class="composeEntry" :class="{ multipleReferences: currentReferences.length > 1 }">
                <div class="attachments" aria-label="参考素材">
                  <div v-for="(item, index) in currentReferences" :key="item.url" class="reference">
                    <img :src="item.url" :alt="referenceLabel(index)" />
                    <button
                      v-if="mode === 'image' && referenceLimit > 1"
                      type="button"
                      class="referenceLabel"
                      :aria-label="`插入参考图编号 ${index + 1}`"
                      :title="`插入[图${index + 1}]`"
                      @mousedown.prevent
                      @click="insertReferenceToken(index)"
                    >
                      [图{{ index + 1 }}]
                    </button>
                    <span v-else class="referenceLabel">{{ referenceLabel(index) }}</span>
                    <button type="button" class="removeReference" :aria-label="`移除${referenceLabel(index)}`" @click="removeReference(index)"><icon-x :size="15" /></button>
                  </div>
                  <button v-if="currentReferences.length < referenceLimit" class="upload" type="button" @click="openReferencePicker">
                    <icon-photo-plus :size="19" /> {{ mode === "video" ? "起始画面" : "添加参考图" }}
                  </button>
                </div>
                <textarea ref="promptInput" v-model="prompt" class="composePrompt" rows="2" maxlength="10000" aria-label="提示词" :placeholder="promptPlaceholder" @paste="pasteReference" />
              </div>
              <div class="composeFooter">
                <div class="composeOptions">
                  <span><icon-photo v-if="mode === 'image'" :size="17" /><icon-video v-else :size="17" />{{ choices[mode]?.label || (mode === "image" ? "Muse Image" : "Muse Video") }}</span>
                  <parameterMenu v-model="ratio" label="画面比例" :options="ratiosFor(mode)" :icon="IconAspectRatio" />
                  <parameterMenu v-if="mode === 'video'" v-model="duration" label="时长" :options="quickDurations" :icon="IconClock" unit="秒" />
                </div>
                <div class="composeAction">
                  <span class="composeNote" role="status" id="generationStatus">{{ statusLabel }}</span>
                  <router-link v-if="latest && !generating && !onCreate" class="pillButton resultButton" to="/studio/create">查看结果</router-link>
                  <button v-if="pendingTask && !generating && !onCreate" class="pillButton resultButton" type="button" @click="resume">查询任务</button>
                  <button class="sendButton" type="button" :disabled="!canGenerate" aria-label="生成" aria-describedby="generationStatus" @click="submitGeneration"><icon-arrow-up :size="23" /></button>
                </div>
              </div>
              <p v-if="referenceError || (!onCreate && generationError)" class="errorNote" role="alert">{{ referenceError || generationError }}</p>
            </div>
          </div>
          <div class="composerMini" :inert="!composerCollapsed">
            <button ref="miniButton" class="miniButton" type="button" aria-label="展开生成框" @click="expandComposer">
              <span class="miniIcon"><icon-sparkles :size="17" /></span>
              <span class="miniLabel">{{ statusLabel || prompt.trim() || (currentReferences.length ? "继续编辑参考图" : "描述你想创作的画面…") }}</span>
              <icon-chevron-up :size="17" aria-hidden="true" />
            </button>
          </div>
        </section>
      </template>

      <template v-else-if="route.name === 'studioWorks'">
        <div class="libraryHeading">
          <div><span class="stageEyebrow">我的创作</span><h1>资产库</h1></div>
          <router-link class="darkButton" to="/studio/create"><icon-plus :size="17" />新建创作</router-link>
        </div>
        <div v-if="works.length" class="libraryToolbar">
          <div class="libraryTabs" role="group" aria-label="作品分类">
            <button v-for="option in workFilters" :key="option.id" type="button" :aria-pressed="workFilter === option.id" @click="workFilter = option.id">
              <icon-star v-if="option.id === 'favorite'" :size="15" aria-hidden="true" />
              {{ option.label }} <small>{{ workCounts[option.id] }}</small>
            </button>
          </div>
          <div class="libraryTools">
            <label class="librarySearch"><icon-search :size="17" /><input v-model.trim="workQuery" type="search" aria-label="搜索作品" placeholder="搜索名称、标签或提示词" /></label>
            <parameterMenu v-model="workSort" label="排序" :options="sortOptions" :icon="IconArrowsSort" />
            <button class="libraryButton" type="button" :aria-pressed="selectingWorks" :disabled="packingWorks" @click="toggleWorkSelection">
              <icon-list-check :size="16" aria-hidden="true" />{{ selectingWorks ? "退出多选" : "多选" }}
            </button>
          </div>
          <div v-if="libraryTags.length || workTag" class="libraryTags" role="group" aria-label="按标签筛选">
            <icon-tag :size="16" aria-hidden="true" />
            <button type="button" :aria-pressed="!workTag" @click="workTag = ''">全部标签</button>
            <button v-for="tag in libraryTags" :key="tag" type="button" :aria-pressed="workTag === tag" @click="workTag = workTag === tag ? '' : tag">{{ tag }}</button>
          </div>
        </div>
        <div v-if="selectingWorks" class="selectionBar">
          <span role="status">已选 <strong>{{ selectedWorks.length }}</strong> 件<small v-if="hiddenSelectedCount"> · {{ hiddenSelectedCount }} 件在筛选外</small></span>
          <div class="selectionActions">
            <button class="libraryButton" type="button" :disabled="packingWorks || !shownWorks.length || everyShownWorkSelected" @click="selectShownWorks">全选当前</button>
            <button class="libraryButton" type="button" :disabled="packingWorks || !selectedWorks.length" @click="selectedWorkPaths.clear()">清空</button>
            <button class="darkButton" type="button" :disabled="packingWorks || !selectedWorks.length" @click="downloadSelectedWorks"><icon-download :size="16" aria-hidden="true" />{{ packingWorks ? "正在打包…" : "下载 ZIP" }}</button>
          </div>
        </div>
        <p v-if="archiveError" class="errorNote" role="alert">{{ archiveError }}</p>
        <p v-if="loadingWorks && !works.length" role="status">正在读取作品…</p>
        <p v-if="workError" class="errorNote" role="alert">{{ workError }} <button class="pillButton" type="button" @click="loadWorks">重试</button></p>
        <div v-if="shownWorks.length" class="libraryGrid">
          <article v-for="work in shownWorks" :key="work.path" class="assetCard" :class="{ assetSelected: selectingWorks && selectedWorkPaths.has(work.path) }">
            <div class="assetMedia">
              <button class="assetPreview" type="button" :aria-label="`${selectingWorks ? '选择' : '预览'} ${workTitle(work)}`"
                :aria-pressed="selectingWorks ? selectedWorkPaths.has(work.path) : undefined" @click="selectingWorks ? selectWork(work.path) : openWork(work)">
                <workCover :directory="directory" :work="work" />
                <span v-if="selectingWorks" class="assetSelect" aria-hidden="true"><icon-check v-if="selectedWorkPaths.has(work.path)" :size="16" /></span>
              </button>
              <button v-if="!selectingWorks" class="assetFavorite" type="button" :aria-label="`${work.details?.favorite ? '取消收藏' : '收藏'} ${workTitle(work)}`"
                :aria-pressed="Boolean(work.details?.favorite)" :disabled="savingWorkPaths.has(work.path)" @click="toggleFavorite(work)"><icon-star :size="17" aria-hidden="true" /></button>
            </div>
            <div class="assetDetails">
              <div class="assetText">
                <strong :title="workTitle(work)">{{ workTitle(work) }}</strong>
                <div class="assetMeta">
                  <time v-if="work.modifiedAt" :datetime="new Date(work.modifiedAt).toISOString()">{{ dateFormat.format(work.modifiedAt) }}</time>
                  <button v-if="work.details?.tags?.[0] && !selectingWorks" class="assetTag" type="button" :title="work.details.tags[0]" @click="workTag = work.details.tags[0]">{{ work.details.tags[0] }}</button>
                </div>
              </div>
              <button v-if="!selectingWorks" class="assetAction" type="button" :aria-label="`下载 ${workTitle(work)}`" title="下载原件" @click="downloadWork(work)"><icon-download :size="17" /></button>
            </div>
          </article>
        </div>
        <div v-else-if="works.length && !loadingWorks" class="emptyState">
          <span class="emptyIcon"><icon-search :size="25" /></span>
          <h2>{{ workFilter === "favorite" && !workCounts.favorite ? "还没有收藏" : "没有匹配的素材" }}</h2>
          <button class="pillButton" type="button" @click="clearWorkFilters">查看全部作品</button>
        </div>
        <section v-else-if="!loadingWorks && !workError" class="emptyState" aria-label="作品库">
          <span class="emptyIcon"><icon-photo :size="27" /></span>
          <h2>暂无作品</h2>
          <p>快速创作生成的图片和视频会保存在这里。</p>
          <router-link class="darkButton" to="/studio">前往快速创作 <icon-arrow-right :size="17" /></router-link>
        </section>
      </template>

      <template v-else-if="route.name === 'studioProjects'">
        <div class="pageHeading withAction">
          <div><h1>短剧工作台</h1><p>管理剧本、分镜与项目</p></div>
          <button class="darkButton" type="button" :disabled="busy" @click="showNewProject"><icon-plus :size="19" />新建项目</button>
        </div>
        <div class="sectionHeading">
          <h2>我的项目 <small v-if="projects.length">{{ projects.length }}</small></h2>
          <router-link v-if="!publicWeb" class="textLink" to="/home">原版工作台 <icon-arrow-up-right :size="15" /></router-link>
        </div>
        <p v-if="projectError" class="errorNote" role="alert">{{ projectError }}</p>
        <div v-if="projects.length" class="projectGrid">
          <button v-for="project in projects" :key="project.directory" class="projectCard" type="button" :disabled="busy" @click="openProject(project.directory)">
            <projectCover :directory="project.directory" />
            <span class="projectInfo">
              <strong>{{ project.name }}</strong>
              <time :datetime="new Date(project.lastOpenedAt).toISOString()">最近打开 · {{ dateFormat.format(project.lastOpenedAt) }}</time>
            </span>
            <icon-arrow-up-right class="projectArrow" :size="18" aria-hidden="true" />
          </button>
        </div>
        <section v-else class="emptyState" aria-label="项目列表">
          <span class="emptyIcon"><icon-movie :size="28" /></span>
          <h2>还没有项目</h2>
          <p>新建项目后，在现有工作区继续创作。</p>
        </section>
      </template>

      <template v-else>
        <div class="pageHeading"><h1>设置</h1></div>
        <div class="settingsGrid">
          <nav class="settingsNav" aria-label="设置导航">
            <button v-if="currentUser" :aria-pressed="settingsTab === 'account'" type="button" @click="settingsTab = 'account'"><icon-user :size="18" />账号</button>
            <button :aria-pressed="settingsTab === 'models'" type="button" @click="settingsTab = 'models'"><icon-sparkles :size="18" />创作模型</button>
          </nav>
          <section class="settingsPanel">
            <template v-if="settingsTab === 'account' && currentUser">
              <h2>账号</h2>
              <userAccount :request="http" :userName="currentUser.userName" :role="currentUser.role" @loggedOut="reload" />
            </template>
            <template v-else>
              <h2>创作模型</h2>
              <p>模型由网站统一提供，你无需配置接口、密钥或运行后端。</p>
              <div v-for="option in modes" :key="option.id" class="modelRow">
                <span class="modelIcon"><component :is="option.icon" :size="22" /></span>
                <span><strong>{{ option.id === "image" ? "Muse Image" : "Muse Video" }}</strong><small>{{ option.label }}生成</small></span>
                <span class="modelStatus">{{ catalogLoading ? "读取中" : choices[option.id] ? "已接入" : "未开放" }}</span>
              </div>
            </template>
          </section>
        </div>
      </template>
    </main>

    <dialog ref="inspirationDialog" class="detailDialog inspirationDialog" aria-labelledby="inspirationTitle" @click.self="inspirationDialog?.close()" @close="selected = undefined">
      <article v-if="selected" class="inspirationDetail">
        <figure class="inspirationArt"><img :src="selected.url" :alt="selected.title" /></figure>
        <button class="recipeClose" type="button" aria-label="关闭" autofocus @click="inspirationDialog?.close()"><icon-x :size="18" aria-hidden="true" /></button>
        <div class="inspirationRecipe">
          <header class="recipeTitle">
            <span>{{ selected.category }}</span>
            <h2 id="inspirationTitle">{{ selected.title }}</h2>
          </header>
          <section class="recipePrompt" :aria-label="recipeMode === 'image' ? '画面提示词' : '运镜提示词'">
            <div class="recipeToolbar">
              <div v-if="selected.motion" class="recipeTabs" role="group" aria-label="提示词类型">
                <button type="button" :aria-pressed="recipeMode === 'image'" @click="selectRecipe('image')">画面提示词</button>
                <button type="button" :aria-pressed="recipeMode === 'video'" @click="selectRecipe('video')">运镜提示词</button>
              </div>
              <span v-else class="recipeLabel">画面提示词</span>
              <button class="recipeCopy" type="button" aria-label="复制提示词" @click="copyPrompt">
                <icon-check v-if="copied" :size="15" aria-hidden="true" /><icon-copy v-else :size="15" aria-hidden="true" />
                <span role="status">{{ copied ? "已复制" : "复制" }}</span>
              </button>
            </div>
            <p class="recipeText">{{ recipePrompt }}</p>
            <p v-if="copyFailed" class="recipeCopyError" role="status">剪贴板不可用，可选中文本手动复制。</p>
          </section>
          <footer class="recipeActions">
            <button type="button" :class="{ primaryRecipeAction: recipeMode === 'image' }" @click="useInspiration('image')"><icon-photo-plus :size="17" aria-hidden="true" />用作参考图</button>
            <button type="button" :class="{ primaryRecipeAction: recipeMode === 'video' }" @click="useInspiration('video')"><icon-player-play :size="17" aria-hidden="true" />让它动起来</button>
          </footer>
        </div>
      </article>
    </dialog>

    <dialog ref="workDialog" class="detailDialog workDialog" :class="{ editingDetails: editingWork }" aria-labelledby="workTitle" @click.self="dismissWork" @cancel="editingWork && viewedWork && savingWorkPaths.has(viewedWork.path) && $event.preventDefault()" @close="closeWork()">
      <article v-if="viewedWork" class="detailBody">
        <header class="workDetailHeader">
          <h2 id="workTitle">{{ editingWork ? "编辑作品资料" : workTitle(viewedWork) }}</h2>
          <div class="workDetailTools">
            <button v-if="!editingWork" class="libraryButton" type="button" :disabled="savingWorkPaths.has(viewedWork.path)" @click="editWork(viewedWork)"><icon-pencil :size="16" aria-hidden="true" />编辑资料</button>
            <button v-if="!editingWork" class="assetAction" type="button" :aria-label="viewedWork.details?.favorite ? '取消收藏' : '收藏作品'"
              :aria-pressed="Boolean(viewedWork.details?.favorite)" :disabled="savingWorkPaths.has(viewedWork.path)" @click="toggleFavorite(viewedWork)"><icon-star :size="17" aria-hidden="true" /></button>
            <button class="iconButton" type="button" aria-label="关闭" autofocus :disabled="editingWork && savingWorkPaths.has(viewedWork.path)" @click="dismissWork"><icon-x :size="19" /></button>
          </div>
        </header>
        <form v-if="editingWork" class="workEditForm" @submit.prevent="saveWorkEdit(viewedWork)">
          <label for="assetName">作品名称</label>
          <input id="assetName" ref="workNameInput" v-model.trim="workNameDraft" type="text" maxlength="60" placeholder="给作品起个名字" />
          <label for="assetTags">标签</label>
          <input id="assetTags" v-model.trim="workTagsDraft" type="text" maxlength="160" aria-describedby="assetTagsHelp" placeholder="例如：产品摄影，雨夜" />
          <small id="assetTagsHelp">用逗号分隔，最多 8 个标签，每个最多 16 个字。</small>
          <p v-if="workActionError" class="errorNote" role="alert">{{ workActionError }}</p>
          <footer>
            <button class="pillButton" type="button" :disabled="savingWorkPaths.has(viewedWork.path)" @click="editingWork = false; workActionError = ''">取消</button>
            <button class="darkButton" type="submit" :disabled="savingWorkPaths.has(viewedWork.path)">{{ savingWorkPaths.has(viewedWork.path) ? "保存中…" : "保存" }}</button>
          </footer>
        </form>
        <template v-else>
        <p v-if="previewError" class="errorNote" role="alert">{{ previewError }}</p>
        <p v-else-if="!workUrl" role="status">正在加载作品…</p>
        <img v-else-if="viewedWork.mediaType === 'image'" class="hero" :src="workUrl" alt="生成图片" @error="previewError = '暂时无法读取这件作品。'" />
        <video v-else class="hero" :src="workUrl" controls playsinline aria-label="生成视频" @error="previewError = '暂时无法播放这件作品。'" />
        <div class="recipeFacts">
          <span>{{ viewedWork.mediaType === "image" ? "图片" : "视频" }}</span>
          <span v-if="viewedWork.ratio">{{ viewedWork.ratio }}</span>
          <span v-if="viewedWork.duration">{{ viewedWork.duration }} 秒</span>
          <span v-if="viewedWork.images?.length">{{ viewedWork.images.length }} 张参考图</span>
        </div>
        <div v-if="viewedWork.details?.tags?.length" class="workDetailTags" aria-label="作品标签"><span v-for="tag in viewedWork.details.tags" :key="tag">{{ tag }}</span></div>
        <p v-if="viewedWork.prompt" class="promptText">{{ viewedWork.prompt }}</p>
        <p v-if="referenceError" class="errorNote" role="alert">{{ referenceError }}</p>
        <p v-if="workActionError" class="errorNote" role="alert">{{ workActionError }}</p>
        <footer v-if="workUrl">
          <a class="pillButton" :href="workUrl" :download="workFileName(viewedWork)">下载</a>
          <button v-if="viewedWork.prompt" class="pillButton" type="button" @click="reuseWork(viewedWork)">再次编辑</button>
          <button v-if="canUseAsReference(viewedWork)" class="pillButton" type="button" @click="referenceWork(viewedWork, 'image')">用作参考图</button>
          <button v-if="canUseAsReference(viewedWork)" class="darkButton" type="button" @click="referenceWork(viewedWork, 'video')"><icon-player-play :size="17" />让它动起来</button>
        </footer>
        </template>
      </article>
    </dialog>

    <dialog ref="referenceDialog" class="detailDialog referenceDialog" aria-labelledby="referenceTitle" @click.self="referenceDialog?.close()">
      <div class="referencePicker">
        <header>
          <div><span class="stageEyebrow">参考素材</span><h2 id="referenceTitle">{{ mode === "video" ? "选择起始画面" : "添加参考图" }}</h2></div>
          <button class="iconButton" type="button" aria-label="关闭" autofocus @click="referenceDialog?.close()"><icon-x :size="19" /></button>
        </header>
        <label class="pickerUpload">
          <span class="pickerUploadIcon"><icon-photo-plus :size="23" /></span>
          <span><strong>上传本地图片</strong><small>JPG、PNG、WebP，每张最多 10 MB</small></span>
          <icon-plus :size="19" />
          <input type="file" :accept="quickImageTypes.join(',')" :multiple="referenceLimit > 1" aria-label="上传本地图片" @change="selectReference" />
        </label>
        <p v-if="referenceError" class="errorNote" role="alert">{{ referenceError }}</p>
        <div class="pickerHeading"><h3>从我的作品选择</h3><span>{{ currentReferences.length }} / {{ referenceLimit }} 已添加</span></div>
        <p v-if="workError" class="errorNote" role="alert">{{ workError }} <button class="pillButton" type="button" @click="loadWorks">重试</button></p>
        <p v-else-if="loadingWorks && !referenceWorks.length" role="status">正在读取作品…</p>
        <p v-else-if="!referenceWorks.length" class="pickerEmpty">生成的图片会出现在这里。</p>
        <div v-else class="pickerGrid">
          <button v-for="work in referenceWorks" :key="work.path" class="pickerCard" type="button" :aria-label="workTitle(work)" :aria-pressed="hasReference(work.path)"
            :disabled="!hasReference(work.path) && currentReferences.length >= referenceLimit" @click="toggleReferenceWork(work)">
            <workCover :directory="directory" :work="work" />
            <span class="pickerCheck"><icon-check v-if="hasReference(work.path)" :size="16" /><icon-plus v-else :size="16" /></span>
            <span :title="workTitle(work)">{{ workTitle(work) }}</span>
          </button>
        </div>
        <footer><button class="darkButton" type="button" @click="referenceDialog?.close()">完成</button></footer>
      </div>
    </dialog>

    <dialog ref="projectDialog" class="newDialog" aria-labelledby="newProjectTitle" @cancel="busy && $event.preventDefault()">
      <form class="detailBody" @submit.prevent="createProject">
        <h2 id="newProjectTitle">新建项目</h2>
        <label for="newProjectName">项目名称</label>
        <input id="newProjectName" v-model.trim="projectName" maxlength="80" required autofocus :disabled="busy" placeholder="给你的短剧起个名字" />
        <p v-if="projectError" class="errorNote" role="alert">{{ projectError }}</p>
        <footer><button class="pillButton" type="button" :disabled="busy" @click="projectDialog?.close()">取消</button><button class="darkButton" :disabled="busy || !projectName" type="submit">{{ busy ? "创建中…" : "创建项目" }}</button></footer>
      </form>
    </dialog>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from "vue";
import { useRoute, useRouter } from "vue-router";
import { IconArrowRight, IconArrowUp, IconArrowUpRight, IconArrowsSort, IconAspectRatio, IconCheck, IconChevronDown, IconChevronUp, IconClock, IconCopy, IconDownload, IconListCheck, IconMovie, IconPencil, IconPhoto, IconPhotoPlus, IconPlayerPlay, IconPlus, IconSearch, IconSettings, IconSparkles, IconStar, IconTag, IconUser, IconVideo, IconX } from "@tabler/icons-vue";
import { ElMessage } from "element-plus";
import userAccount from "@toonflow/multi-tenant-user-web/account";
import projectCover from "@/pages/home/projectCover.vue";
import http from "@/lib/http";
import saveFile from "@/lib/saveFile";
import useWorkspaceFiles from "@/lib/workspaceFiles";
import { writeClipboardText } from "@/lib/clipboard";
import { currentUser, publicWeb } from "@/stores/multiTenant";
import { useWorkspaceStore } from "@/stores/workspace";
import { categories, inspirations, type Inspiration, type InspirationCategory } from "./inspirations";
import useQuickMedia, { quickDurations, quickImageSize, quickImageTypes, type QuickMode, type QuickReference, type QuickWork } from "./useQuickMedia";
import workCover from "./workCover.vue";
import { downloadWorkArchive, workFileName } from "./workDownload";
import parameterMenu from "./parameterMenu.vue";
import bookshop from "./bookshop.webp";
import teaTin from "./teaTin.webp";
import littleSub from "./littleSub.webp";

const route = useRoute();
const router = useRouter();
const onCreate = computed(() => route.name === "studioCreate");
const showComposer = computed(() => route.name === "studio" || onCreate.value);
const workspaceStore = useWorkspaceStore();
const navigation = [
  { name: "studioWorks", label: "我的作品", icon: IconPhoto },
  { name: "studio", label: "快速创作", icon: IconSparkles },
  { name: "studioProjects", label: "短剧工作台", icon: IconMovie },
];
const modes = [{ id: "image", label: "图片", icon: IconPhoto }, { id: "video", label: "视频", icon: IconVideo }] as const;
const workFilters = [{ id: "all", label: "全部" }, ...modes, { id: "favorite", label: "收藏" }] as const;
const { directory, choices, ratiosFor, referenceLimitFor, works, loadingWorks, catalogLoading, workError, generationError, busy: generating, activeMode, latest, pendingTask, loadCatalog, loadWorks, saveWorkDetails, run, restore, resume } = useQuickMedia();
const workFilter = ref<(typeof workFilters)[number]["id"]>("all");
const sortOptions = ["最新优先", "最早优先"] as const;
const workSort = ref<(typeof sortOptions)[number]>("最新优先");
const workQuery = ref("");
const workTag = ref("");
const libraryTags = computed(() => [...new Set(works.value.flatMap(work => work.details?.tags ?? []))].sort((left, right) => left.localeCompare(right, "zh-CN")));
const workCounts = computed(() => {
  const image = works.value.filter(work => work.mediaType === "image").length;
  const favorite = works.value.filter(work => work.details?.favorite).length;
  return { all: works.value.length, image, video: works.value.length - image, favorite };
});
const shownWorks = computed(() => {
  const terms = workQuery.value.toLowerCase().split(/\s+/).filter(Boolean);
  const filtered = works.value.filter(work => {
    if (workFilter.value === "favorite" ? !work.details?.favorite : workFilter.value !== "all" && work.mediaType !== workFilter.value) return false;
    if (workTag.value && !work.details?.tags?.includes(workTag.value)) return false;
    const type = work.mediaType === "image" ? "图片" : "视频";
    const searchText = [type, workTitle(work), work.path, work.prompt, ...(work.details?.tags ?? [])].join(" ").toLowerCase();
    return terms.every(term => searchText.includes(term));
  });
  return workSort.value === "最早优先" ? filtered.reverse() : filtered;
});
const selectingWorks = ref(false);
const selectedWorkPaths = ref(new Set<string>());
const selectedWorks = computed(() => works.value.filter(work => selectedWorkPaths.value.has(work.path)));
const everyShownWorkSelected = computed(() => shownWorks.value.every(work => selectedWorkPaths.value.has(work.path)));
const hiddenSelectedCount = computed(() => selectedWorks.value.length - shownWorks.value.filter(work => selectedWorkPaths.value.has(work.path)).length);
const packingWorks = ref(false);
const archiveError = ref("");
const savingWorkPaths = ref(new Set<string>());
const workDialog = ref<HTMLDialogElement>();
const viewedWorkPath = ref("");
const viewedWork = computed(() => works.value.find(work => work.path === viewedWorkPath.value));
const workUrl = ref("");
const previewError = ref("");
const editingWork = ref(false);
const workNameInput = ref<HTMLInputElement>();
const workNameDraft = ref("");
const workTagsDraft = ref("");
const workActionError = ref("");
const activeCategory = ref<InspirationCategory>("全部");
const inspirationQuery = ref("");
const exploreResults = ref<HTMLElement>();
const featured = [
  { title: "镜头里的故事", subtitle: "雨夜、长镜头与电影布光", category: "电影感", image: bookshop },
  { title: "静物研究所", subtitle: "电商主图与棚拍质感", category: "产品摄影", image: teaTin },
  { title: "小小异世界", subtitle: "手工定格、绘本与动画", category: "动画插画", image: littleSub },
] as const;
const visibleInspirations = computed(() => {
  const words = inspirationQuery.value.toLowerCase().split(/\s+/).filter(Boolean);
  return inspirations.filter(item => (activeCategory.value === "全部" || item.category === activeCategory.value)
    && (!words.length || words.every(word => `${item.title} ${item.category} ${item.prompt} ${item.motion ?? ""}`.toLowerCase().includes(word))));
});
const mode = ref<QuickMode>("image");
const prompt = ref("");
const ratiosByMode = ref<Record<QuickMode, string>>({ image: "16:9", video: "16:9" });
const ratio = computed({
  get: () => {
    const options = ratiosFor(mode.value);
    const selected = ratiosByMode.value[mode.value];
    return options.includes(selected) ? selected : options[0];
  },
  set: value => { ratiosByMode.value[mode.value] = value; },
});
const duration = ref(6);
const referencesByMode = ref<Record<QuickMode, QuickReference[]>>({ image: [], video: [] });
const currentReferences = computed(() => referencesByMode.value[mode.value]);
const referenceLimit = computed(() => referenceLimitFor(mode.value));
const promptPlaceholder = computed(() => mode.value === "video" ? "描述主体的动作、镜头和光线变化…" :
  currentReferences.value.length > 1 ? "用 [图1]、[图2] 指定参考用途，例如：保留 [图1] 的人物，参考 [图2] 的光线。" : "描述你想生成的图片…");
const referenceDialog = ref<HTMLDialogElement>();
const referenceError = ref("");
const submittedPrompt = ref("");
const stagePrompt = computed(() => latest.value?.prompt || pendingTask.value?.prompt || submittedPrompt.value);
const stageUrl = ref("");
const stageError = ref("");
const stageTitle = computed(() => generating.value ? (activeMode.value === "video" ? "正在生成视频" : "正在生成图片")
  : pendingTask.value ? "任务待确认"
  : generationError.value || stageError.value ? "这次没有拿到作品"
  : latest.value ? "作品已生成" : "准备开始创作");
const canGenerate = computed(() => !generating.value && !pendingTask.value && !!choices.value[mode.value] && !!prompt.value.trim() && currentReferences.value.length <= referenceLimit.value);
const statusLabel = computed(() => generating.value ? (activeMode.value === "video" ? "视频生成中…" : "图片生成中…")
  : pendingTask.value ? "任务待确认"
  : latest.value ? "生成完成"
  : catalogLoading.value ? "正在读取模型"
  : !choices.value[mode.value] ? "模型未开放" : "");
const promptInput = ref<HTMLTextAreaElement>();
const composer = ref<HTMLElement>();
const miniButton = ref<HTMLButtonElement>();
const composerCollapsed = ref(false);
const inspirationDialog = ref<HTMLDialogElement>();
const selected = ref<Inspiration>();
const recipeMode = ref<QuickMode>("image");
const recipePrompt = computed(() => recipeMode.value === "video" && selected.value?.motion ? selected.value.motion : selected.value?.prompt ?? "");
const copied = ref(false);
const copyFailed = ref(false);
const settingsTab = ref(currentUser.value ? "account" : "models");
const projectDialog = ref<HTMLDialogElement>();
const projectName = ref("");
const projectError = ref("");
const busy = ref(false);
const projects = computed(() => [...workspaceStore.projectList].sort((left, right) => right.lastOpenedAt - left.lastOpenedAt));
const dateFormat = new Intl.DateTimeFormat("zh-CN", { dateStyle: "medium", timeStyle: "short" });
const referenceWorks = computed(() => works.value.filter(canUseAsReference));

let browsing = false;
function updateDock() {
  if (route.name !== "studio") return;
  const scrollTop = window.scrollY;
  const next = scrollTop > (browsing ? 64 : 128);
  if (next === browsing) return;
  browsing = next;
  if (!composer.value?.contains(document.activeElement)) composerCollapsed.value = next;
}

onMounted(() => {
  updateDock();
  window.addEventListener("scroll", updateDock, { passive: true });
  void loadCatalog();
  restore();
  if (generating.value && route.name === "studio") void router.replace("/studio/create");
});
watch([latest, directory], async ([work, root], _previous, onCleanup) => {
  stageUrl.value = "";
  stageError.value = "";
  if (!work || !root) return;
  const preview = useWorkspaceFiles(root).acquireUrl(work.path, work.mimeType);
  let stale = false;
  onCleanup(() => { stale = true; preview.release(); });
  try {
    const url = await preview.url;
    if (!stale) stageUrl.value = url;
  } catch {
    if (!stale) stageError.value = "文件已保存，但暂时无法预览，请到「我的作品」重试。";
  }
});
watch(() => route.name, name => { if (name === "studioWorks") void loadWorks(); }, { immediate: true });
watch(() => route.name, () => {
  browsing = false;
  composerCollapsed.value = false;
  updateDock();
}, { flush: "post" });
onBeforeUnmount(() => {
  window.removeEventListener("scroll", updateDock);
  replaceReferences("image", []);
  replaceReferences("video", []);
  closeWork();
});

async function minimizeComposer() {
  composerCollapsed.value = true;
  await nextTick();
  miniButton.value?.focus({ preventScroll: true });
}

async function expandComposer() {
  composerCollapsed.value = false;
  await nextTick();
  promptInput.value?.focus({ preventScroll: true });
}

async function submitGeneration() {
  if (!canGenerate.value) return;
  const text = prompt.value.trim();
  submittedPrompt.value = text;
  const task = run(mode.value, text, ratio.value, duration.value, [...currentReferences.value]);
  await router.push("/studio/create");
  window.scrollTo(0, 0);
  await task;
}

async function openInspiration(item: Inspiration) {
  selected.value = item;
  selectRecipe("image");
  await nextTick();
  if (!inspirationDialog.value?.open) inspirationDialog.value?.showModal();
}

function selectRecipe(type: QuickMode) {
  recipeMode.value = type;
  copied.value = false;
  copyFailed.value = false;
}

function resetInspirationFilters() {
  activeCategory.value = "全部";
  inspirationQuery.value = "";
}

function showCollection(category: InspirationCategory) {
  activeCategory.value = category;
  inspirationQuery.value = "";
  exploreResults.value?.scrollIntoView({ behavior: window.matchMedia("(prefers-reduced-motion: reduce)").matches ? "instant" : "smooth" });
}

async function copyPrompt() {
  const item = selected.value;
  if (!item) return;
  const recipe = recipeMode.value;
  try {
    await writeClipboardText(recipePrompt.value);
    if (selected.value === item && recipeMode.value === recipe) { copied.value = true; copyFailed.value = false; }
  } catch {
    if (selected.value === item && recipeMode.value === recipe) copyFailed.value = true;
  }
}

function changeMode(type: QuickMode) {
  mode.value = type;
  referenceError.value = "";
}

function referenceLabel(index: number) {
  return mode.value === "video" ? "起始画面" : `参考图 ${index + 1}`;
}

async function insertReferenceToken(index: number) {
  const input = promptInput.value;
  if (!input || mode.value !== "image" || !currentReferences.value[index]) return;
  const token = `[图${index + 1}]`;
  const position = input.selectionEnd;
  if (input.maxLength >= 0 && prompt.value.length + token.length > input.maxLength) {
    ElMessage.warning("提示词已达长度上限，无法插入参考图编号。");
    return;
  }
  prompt.value = prompt.value.slice(0, position) + token + prompt.value.slice(position);
  await nextTick();
  input.focus({ preventScroll: true });
  input.setSelectionRange(position + token.length, position + token.length);
}

function replaceReferences(type: QuickMode, next: QuickReference[]) {
  for (const item of referencesByMode.value[type]) {
    if (item.file && !next.some(reference => reference.url === item.url)) URL.revokeObjectURL(item.url);
  }
  referencesByMode.value[type] = next;
}

function removeReference(index: number) {
  replaceReferences(mode.value, currentReferences.value.filter((_, position) => position !== index));
  referenceError.value = "";
}

function appendReference(type: QuickMode, reference: QuickReference) {
  const previous = referencesByMode.value[type];
  if (previous.some(item => item.url === reference.url)) return true;
  if (previous.length >= referenceLimitFor(type)) return false;
  referencesByMode.value[type] = [...previous, reference];
  return true;
}

function addLocalFiles(files: File[]) {
  referenceError.value = "";
  for (const file of files) {
    if (!quickImageTypes.includes(file.type) || !file.size || file.size > quickImageSize) {
      referenceError.value = "部分图片无法添加。请选择不超过 10 MB 的 JPG、PNG 或 WebP。";
      continue;
    }
    if (currentReferences.value.some(({ file: existing }) => existing?.name === file.name && existing.size === file.size && existing.lastModified === file.lastModified)) continue;
    if (currentReferences.value.length >= referenceLimit.value) {
      referenceError.value = `当前模式最多使用 ${referenceLimit.value} 张参考图。`;
      break;
    }
    appendReference(mode.value, { url: URL.createObjectURL(file), file });
  }
  if (!referenceError.value && currentReferences.value.length >= referenceLimit.value) referenceDialog.value?.close();
}

function selectReference(event: Event) {
  const input = event.target as HTMLInputElement;
  addLocalFiles(Array.from(input.files ?? []));
  input.value = "";
}

function pasteReference(event: ClipboardEvent) {
  const files = Array.from(event.clipboardData?.files ?? []);
  if (!files.some(file => file.type.startsWith("image/"))) return;
  event.preventDefault();
  addLocalFiles(files);
}

function openReferencePicker() {
  referenceError.value = "";
  referenceDialog.value?.showModal();
  void loadWorks();
}

function hasReference(path: string) {
  return currentReferences.value.some(reference => reference.asset?.path === path);
}

function workspaceReference(asset: { path: string; mimeType: string }): QuickReference {
  return { asset: { path: asset.path, mimeType: asset.mimeType }, url: useWorkspaceFiles(directory.value).readUrl(asset.path) };
}

function canUseAsReference(work: QuickWork) {
  return work.mediaType === "image" && quickImageTypes.includes(work.mimeType);
}

function toggleReferenceWork(work: QuickWork) {
  const index = currentReferences.value.findIndex(reference => reference.asset?.path === work.path);
  if (index >= 0) removeReference(index);
  else if (directory.value) {
    appendReference(mode.value, workspaceReference(work));
    referenceError.value = "";
    if (mode.value === "video") referenceDialog.value?.close();
  }
}

async function useInspiration(type: QuickMode) {
  const inspiration = selected.value;
  if (!inspiration) return;
  const reference = { url: inspiration.url };
  if (type === "video") {
    replaceReferences(type, [reference]);
    if (!prompt.value.trim() && inspiration.motion) prompt.value = inspiration.motion;
  } else if (!appendReference(type, reference)) {
    referenceError.value = `最多使用 ${referenceLimitFor(type)} 张参考图，请移除一张后再添加。`;
    mode.value = type;
    inspirationDialog.value?.close();
    await expandComposer();
    return;
  }
  changeMode(type);
  inspirationDialog.value?.close();
  await expandComposer();
}

async function openWork(work: QuickWork) {
  closeWork();
  referenceError.value = "";
  viewedWorkPath.value = work.path;
  const url = useWorkspaceFiles(directory.value).readUrl(work.path);
  workUrl.value = url;
  await nextTick();
  if (!workDialog.value?.open) workDialog.value?.showModal();
}

function workTitle(work: QuickWork) {
  if (work.details?.name) return work.details.name;
  const text = work.prompt?.trim().split(/\r?\n/)[0];
  if (text) return text.length > 34 ? `${text.slice(0, 34)}…` : text;
  const number = work.path.split("/").at(-1)?.replace(/^(image|video)/, "").slice(0, 6).toUpperCase();
  return `${work.mediaType === "image" ? "图片" : "视频"} · ${number}`;
}

async function downloadWork(work: QuickWork) {
  const files = useWorkspaceFiles(directory.value);
  try {
    await saveFile(async () => new Blob([await files.read(work.path)], { type: work.mimeType }), workFileName(work));
  } catch { ElMessage.error("下载失败，请稍后重试。"); }
}

function clearWorkFilters() {
  workQuery.value = "";
  workTag.value = "";
  workFilter.value = "all";
}

function toggleWorkSelection() {
  selectingWorks.value = !selectingWorks.value;
  selectedWorkPaths.value.clear();
  archiveError.value = "";
}

function selectWork(path: string) {
  if (packingWorks.value) return;
  const selection = selectedWorkPaths.value;
  if (selection.has(path)) selection.delete(path);
  else selection.add(path);
  archiveError.value = "";
}

function selectShownWorks() {
  for (const work of shownWorks.value) selectedWorkPaths.value.add(work.path);
  archiveError.value = "";
}

async function downloadSelectedWorks() {
  const selection = selectedWorks.value;
  if (packingWorks.value || !selection.length || !directory.value) return;
  packingWorks.value = true;
  archiveError.value = "";
  try {
    if (await downloadWorkArchive(directory.value, selection)) ElMessage.success("作品已打包下载。");
  } catch (error) {
    archiveError.value = error instanceof Error && error.message.startsWith("所选作品") ? error.message : "暂时无法打包作品，请重试。";
  } finally { packingWorks.value = false; }
}

async function toggleFavorite(work: QuickWork) {
  if (savingWorkPaths.value.has(work.path)) return;
  savingWorkPaths.value.add(work.path);
  workActionError.value = "";
  try { await saveWorkDetails(work, { favorite: !work.details?.favorite }); }
  catch {
    const message = "收藏状态保存失败，请重试。";
    if (viewedWorkPath.value === work.path && workDialog.value?.open) workActionError.value = message;
    else ElMessage.error(message);
  } finally { savingWorkPaths.value.delete(work.path); }
}

async function editWork(work: QuickWork) {
  workNameDraft.value = work.details?.name ?? "";
  workTagsDraft.value = work.details?.tags?.join("，") ?? "";
  workActionError.value = "";
  editingWork.value = true;
  await nextTick();
  workNameInput.value?.focus();
}

async function saveWorkEdit(work: QuickWork) {
  if (savingWorkPaths.value.has(work.path)) return;
  const tags = [...new Set(workTagsDraft.value.split(/[,，、]/).map(tag => tag.trim()).filter(Boolean))];
  if (tags.length > 8 || tags.some(tag => tag.length > 16)) {
    workActionError.value = "请保留最多 8 个标签，每个标签不要超过 16 个字。";
    return;
  }
  savingWorkPaths.value.add(work.path);
  workActionError.value = "";
  try {
    await saveWorkDetails(work, { name: workNameDraft.value, tags });
    editingWork.value = false;
  } catch { workActionError.value = "作品资料保存失败，请重试。"; }
  finally { savingWorkPaths.value.delete(work.path); }
  if (!editingWork.value) {
    await nextTick();
    workDialog.value?.querySelector<HTMLButtonElement>(".workDetailTools button")?.focus();
  }
}

function dismissWork() {
  if (editingWork.value && savingWorkPaths.value.has(viewedWorkPath.value)) return;
  workDialog.value?.close();
}

function closeWork() {
  viewedWorkPath.value = "";
  workUrl.value = "";
  previewError.value = "";
  editingWork.value = false;
  workActionError.value = "";
}

async function openDraft() {
  workDialog.value?.close();
  await router.push("/studio/create");
  await expandComposer();
  composer.value?.scrollIntoView({ block: "nearest" });
}

async function reuseWork(work: QuickWork) {
  if (!work.prompt || !directory.value) return;
  changeMode(work.mediaType);
  prompt.value = work.prompt;
  ratiosByMode.value[work.mediaType] = work.ratio ?? "16:9";
  if (work.mediaType === "video") duration.value = work.duration ?? 6;
  replaceReferences(work.mediaType, (work.images ?? []).map(workspaceReference));
  await openDraft();
}

async function referenceWork(work: QuickWork, type: QuickMode) {
  if (!directory.value || !canUseAsReference(work)) return;
  const reference = workspaceReference(work);
  if (type === "video") replaceReferences(type, [reference]);
  else if (!appendReference(type, reference)) {
    referenceError.value = `最多使用 ${referenceLimitFor(type)} 张参考图，请移除一张后再添加。`;
    return;
  }
  changeMode(type);
  await openDraft();
}

function reload() { window.location.reload(); }

function showNewProject() {
  projectName.value = "";
  projectError.value = "";
  projectDialog.value?.showModal();
}

async function openProject(directory: string) {
  if (busy.value) return;
  busy.value = true;
  projectError.value = "";
  try {
    await workspaceStore.openProject(directory);
    await router.push("/workspace");
  } catch (error) {
    projectError.value = error instanceof Error ? error.message : "无法打开项目";
  } finally { busy.value = false; }
}

async function createProject() {
  if (!projectName.value || busy.value) return;
  const name = projectName.value;
  busy.value = true;
  projectError.value = "";
  try {
    const { absolutePath } = await http<{ absolutePath: string }>({ method: "GET", url: "/api/workspaces/list" });
    const folder = `studio-${crypto.randomUUID()}`;
    const files = useWorkspaceFiles(absolutePath);
    await files.mkdir(folder);
    await files.mkdir(`${folder}/assets`);
    await files.writeJson(`${folder}/画布1.json`, { toonflowCanvas: true, nodes: [], edges: [], viewport: { x: 0, y: 0, zoom: 1 } }, true);
    const separator = absolutePath.includes("\\") ? "\\" : "/";
    const directory = absolutePath.endsWith(separator) ? absolutePath + folder : absolutePath + separator + folder;
    await workspaceStore.addProject(directory, name);
    projectDialog.value?.close();
  } catch (error) {
    projectError.value = error instanceof Error ? error.message : "无法创建项目";
  } finally { busy.value = false; }
}
</script>

<style src="./studio.scss" lang="scss" scoped></style>
