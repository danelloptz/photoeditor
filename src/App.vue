<template>
    <div class="modal_wrapper" v-if="isSaveTemplate">
        <div class="modal" :style="{ position: isChangePosition ? 'static' : 'relative' }">
            <img src="@/assets/images/close.png" class="close" @click="closeSaveTemplate">
            <span>Сохранить как шаблон?</span>
            <div class="modal_row">
                <AppBadButton :text="'1'" class="square_btn" @click="saveTemplate(0)"/>
                <AppBadButton :text="'2'" class="square_btn" @click="saveTemplate(1)"/>
                <AppBadButton :text="'3'" class="square_btn" @click="saveTemplate(2)"/>
                <AppBadButton :text="'ОТМЕНА'" class="cancel_btn" @click="closeSaveTemplate"/>
            </div>
        </div>
    </div>
    <div class="wrapper">
        <div class="editor" v-if="windowWidth > 900" @mousedown.self="resetSelect" @touchstart.self="resetSelect">
            <img src="@/assets/images/close.png" class="close" @click="close">
            <h1>Редактор</h1>
            <div class="adding_btns">
                <span>Добавить: </span>
                <AppGoodButton @click="addImage" :text="text6" class="btn1"/>
                <AppGoodButton @click="addTextBlock" :text="text4" class="btn2" />
                <AppGoodButton @click="addRectangle" :text="text5" class="btn1" />
                <AppGoodButton @click="toggleEmojiPanel" :text="'ЭМОДЗИ'" class="btn2" ref="emojiButton" />
                <!-- Панель смайликов -->
                <div v-if="showEmojiPanel" class="emoji-panel" ref="emojiPanel">
                    <span v-for="emoji in emojis" :key="emoji" class="emoji" @click="insertEmoji(emoji)">
                        {{ emoji }}
                    </span>
                </div>
            </div>
            <div class="main">
                <div class="cropper_wrapper">
                    <!-- Контейнер для vue-advanced-cropper -->
                    <div v-if="imageSrc" class="cropper-container" ref="container">
                        <div
                            v-for="block in textBlocks"
                            :key="block.id"
                            class="text-block"
                            :data-id="block.id"
                            :style="{
                                top: `${block.top}px`,
                                left: `${block.left}px`,
                                fontSize: `${block.fontSize}px`,
                                color: block.color,
                                fontWeight: block.fontWeight,
                                fontStyle: block.fontStyle,
                                fontFamily: block.fontFamily,
                                textAlign: block.textAlign, // Выравнивание текста
                                cursor: 'pointer',
                                zIndex: block.zIndex,
                                display: block.display
                            }"
                            @mousedown.stop="selectBlock(block.id, $event)"
                            @touchstart.stop="selectBlock(block.id, $event)"
                        >
                            <!-- Редактируемый текст -->
                            <div
                            :contenteditable="block.id === editingBlockId"
                            @input="updateText(block.id, $event)"
                            @keydown="handleEnterDown($event)"
                            @click.stop="enableEditing(block.id); $event.stopPropagation()"
                            @blur="disableEditing(block.id)"
                            :style="{
                                whiteSpace: 'pre-wrap', // Сохраняем переносы строк
                                cursor: 'pointer',
                                maxHeight: '100%'
                            }"
                            >
                            {{ block.text }}
                            </div>
                        </div>
                        <div
                            v-for="image in images"
                            :key="image.id"
                            class="overlay-image"
                            :style="{
                            top: `${image.top}px`,
                            left: `${image.left}px`,
                            width: `${image.width}px`,
                            height: `${image.height}px`,
                            zIndex: image.zIndex,
                            transform: `rotate(${image.rotation}deg)`,
                            display: image.display,
                            }"
                            @mousedown="selectImage(image.id, $event)"
                            @touchstart="selectImage(image.id, $event)"
                        >
                        <!-- eslint-disable vue/no-use-v-if-with-v-for  -->
                            <div
                                v-if="selectedLay == image.id"
                                v-for="handle in rectangleHandles"
                                :key="handle.position"
                                :class="['handle', handle.position]"
                                @mousedown="startResizeImage($event, handle.position, image.id)"
                                @touchstart="startResizeImage($event, handle.position, image.id)"
                            ></div>
                            <div 
                                v-if="selectedLay == image.id"
                                class="rotate_handle"
                                @mousedown="startRotateImage($event, image.id)"
                                @touchstart="startRotateImage($event, image.id)"
                            >
                                <img src="@/assets/images/return_back.png" />
                            </div>
                            <img
                            :src="image.src"
                            :style="{
                                width: '100%',
                                height: '100%',
                                pointerEvents: 'none'
                            }"
                            />
                        </div>
                        <div
                            v-for="rectangle in rectangles"
                            :key="rectangle.id"
                            class="rectangle"
                            :style="{
                            top: `${rectangle.top}px`,
                            left: `${rectangle.left}px`,
                            width: `${rectangle.width}px`,
                            height: `${rectangle.height}px`,
                            backgroundColor: rectangle.color,
                            opacity: rectangle.opacity,
                            zIndex: rectangle.zIndex,
                            display: rectangle.display
                            }"
                            @mousedown="selectRectangle(rectangle.id, $event)"
                            @touchstart="selectRectangle(rectangle.id, $event)"
                        >
                        <!-- eslint-disable vue/no-use-v-if-with-v-for  -->
                            <div
                                v-if="rectangle.id === selectedLay"
                                v-for="handle in rectangleHandles"
                                :key="handle.position"
                                :class="['handle', handle.position]"
                                @mousedown="startResizeRectangle($event, handle.position, rectangle.id)"
                                @touchstart="startResizeRectangle($event, handle.position, rectangle.id)"
                            ></div>
                        </div>
                        <cropper
                            ref="cropper"
                            class="cropper"
                            :src="currentImage"
                            :resize-image="{
                                wheel: false
                            }"
                            :stencil-props="{
                                handlers: isCroped ? 
                                {
                                    eastNorth: true,
                                    north: false,
                                    westNorth: true,
                                    west: false,
                                    westSouth: true,
                                    south: false,
                                    eastSouth: true,
                                    east: false,
                                } : {},
                                movable: false,
                            }"
                            @change="onCropChange"
                            :style="{
                                maxWidth: windowWidth > 1000 ? '511px' : '400px',
                                background: 'none'
                            }"
                            :default-size="defaultSize"
                            @click="resetSelect"
                        />
                    </div>
                    <div class="crop_msg" v-if="isCroped">
                        <span>Обрезать?</span>
                        <AppGoodButton :text="text7" @click="resize" class="crop" /> 
                    </div>
                    <div class="row row_templates">
                        <span>Шаблон: </span>
                        <AppGoodButton 
                            :isReboot="true"
                            class="square_btn not_active"
                            @click="rebootTemplates"
                        />
                        <AppGoodButton 
                            v-for="(item, index) in ['1', '2', '3']"
                            :text="item" 
                            :key="index"
                            class="square_btn" 
                            :class="{ not_active: index !== selectedTemplate }"
                            @click="selectTemplate(index)"
                        />
                        <AppGoodButton 
                            :isSave="true"
                            class="square_btn not_active"
                            @click="openSaveTemplate"
                        />
                    </div>
                    <div class="row">
                        <AppBadButton :text="text1" class="save" @click="save"/>
                        <AppGoodButton :text="text2" class="download" @click="downloadImage"/>
                    </div>
                    
                </div>
                <div class="tools" @mousedown.self="resetSelect" @touchstart.self="resetSelect">
                    <div class="header_tools">
                        <img src="@/assets/images/crop.png" @click="changeCrop">
                        <img src="@/assets/images/return_back.png" @click="undo">
                        <img src="@/assets/images/return_towards.png" @click="redo">
                    </div>
                    <div class="layers">
                        <h2>Слои:</h2>
                        <div class="line"></div>

                        <draggable 
                            tag="ul"
                            :list="layers"
                            ghost-class="ghost"
                            class="layers-list"
                            @end="onDragEnd" 
                            item-key="name"
                        >
                            <template #item="{element, index}">
                                <div 
                                    class="layer-item"
                                    :class="{ active_lay_item: selectedLay == element.id }"
                                    @click="selectLay(element.id)"
                                >
                                    <img src="@/assets/images/eye.png" v-if="isVisibleLay(element)" class="eye" @click="changeVisible(element)" />
                                    <img src="@/assets/images/close_eye.png" v-if="!isVisibleLay(element)" class="eye" @click="changeVisible(element)" />

                                    <span 
                                        v-if="element.type == 'text'"
                                        class="drag-handle"
                                    >T</span>
                                    <img 
                                        v-if="element.type == 'image' && element?.link"
                                        :src="element?.link"
                                        class="preview"
                                    />
                                    <div 
                                        v-if="element.type == 'rectangle'"
                                        class="preview_rect"
                                    ></div>
                                    <span  
                                        class="layer_name"    
                                    >{{ element.name }}</span>

                                    <img 
                                        class="delete_lay" 
                                        src="@/assets/images/close.png" 
                                        @click="deleteLayer(element, index)"
                                    >
                                </div>
                            </template>
                        </draggable>
                        

                    </div>
                    <div class="text_tools">
                        <div class="text_tools_row">
                            <div class="color_rectangle">
                                <input type="color" class="color_rect" v-model="selectedRectangle.color" @change="captureState" />
                                <h2>Цвет</h2>
                            </div>
                            <div class="vertical_line"></div>
                            <div class="text_text" @click="changeTextStyle('normal')">
                                <span>T</span>
                                <h2>Text</h2>
                            </div>
                            <div class="text_text" @click="changeTextStyle('bold')">
                                <span class="bold">B</span>
                                <h2>Bold</h2>
                            </div>
                            <div class="text_text">
                                <span class="italic" @click="changeTextStyle('italic')">I</span>
                                <h2>Italic</h2>
                            </div>
                            <div class="vertical_line"></div>
                            <div class="color_text">
                                <div 
                                    class="color_circle" 
                                    @click="openColorPicker" 
                                    :style="{ backgroundColor: selectedBlock.color }"
                                ></div>
                                <input 
                                    type="color" 
                                    style="visibility: hidden; height: 0px;" 
                                    ref="colorInput" 
                                    v-model="selectedBlock.color" 
                                    @change="captureState"
                                />
                                <h2>Цвет</h2>
                            </div>
                        </div>
                        <div class="text_tools_row">
                            <div class="rect_opacity">
                                <div
                                    class="rect_op"
                                    :style="{
                                        backgroundColor: `rgba(${hexToRgb(selectedRectangle.color).r}, ${hexToRgb(selectedRectangle.color).g}, ${hexToRgb(selectedRectangle.color).b}, ${selectedRectangle.opacity})`
                                    }"
                                ></div>
                                <h3>Заливка: </h3>
                                <input type="range" class="custom-range custom-range-sm" v-model="selectedRectangle.opacity" min="0" max="1" step="0.01" @change="captureState" />
                            </div>
                            <div class="text_align">
                                <div class="text_align_item" @click="changeTextAling('left')">
                                    <div class="text_align_left" :class="{ non_active_align: selectedBlock.textAlign != 'left' }">
                                        <div></div>
                                        <div></div>
                                        <div></div>
                                        <div></div>
                                    </div>
                                    <h2 :class="{ non_active_align: selectedBlock.textAlign != 'left' }">Left</h2>
                                </div>
                                <div class="text_align_item" @click="changeTextAling('center')">
                                    <div class="text_align_center" :class="{ non_active_align: selectedBlock.textAlign != 'center' }">
                                        <div></div>
                                        <div></div>
                                        <div></div>
                                        <div></div>
                                    </div>
                                    <h2 :class="{ non_active_align: selectedBlock.textAlign != 'center' }">Center</h2>
                                </div>
                                <div class="text_align_item" @click="changeTextAling('right')">
                                    <div class="text_align_right" :class="{ non_active_align: selectedBlock.textAlign != 'right' }">
                                        <div></div>
                                        <div></div>
                                        <div></div>
                                        <div></div>
                                    </div>
                                    <h2 :class="{ non_active_align: selectedBlock.textAlign != 'right' }">Right</h2>
                                </div>
                            </div>
                        </div>
                        <!-- <AppGoodButton :text="text3" class="upload_logo" @click="addImage" /> -->
                        <div class="text_tools_row long">
                            <h2>Размер текста: </h2>
                            <input type="range" class="custom-range custom-range-big" v-model.number="selectedBlock.fontSize" min="5" max="180" step="1" @change="captureState" />
                            <input class="font_size" type="number" v-model="selectedBlock.fontSize" min="5" max="180" @change="captureState" :key="selectedBlock.fontSize" />
                        </div>
                        <div class="text_tools_row">
                            <h2>Шрифт: </h2>
                            <select class="font_dropdown" v-model="selectedBlock.fontFamily">
                                <option value="Tektur" @click="captureState">Tektur</option>
                                <option value="Montserrat" @click="captureState">Montserrat</option>
                                <option value="Arial" @click="captureState">Arial</option>
                                <option value="OpenSans" @click="captureState">Open Sans</option>
                                <option value="Roboto" @click="captureState">Roboto</option>
                            </select>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="editor" v-if="windowWidth <= 900" @mousedown.self="resetSelect" @touchstart.self="resetSelect">
            <img src="@/assets/images/close.png" class="close" @click="close">
            <h1>Редактор</h1>
            <div class="adding_row">
                <span class="adding_btns_text">Добавить:</span>
                <div class="adding_btns">
                    <AppGoodButton 
                        class="square_btn2 not_active"
                        :isText="true"
                        @click="addTextBlock"
                    />
                    <AppGoodButton 
                        class="square_btn2 not_active"
                        :isPics="true"
                        @click="addImage"
                    />
                    <AppGoodButton 
                        class="square_btn2 not_active"
                        :isRect="true"
                        @click="addRectangle"
                    />
                    <AppGoodButton 
                        class="square_btn2 not_active"
                        :isEmojis="true"
                        @click="toggleEmojiPanel"
                        ref="emojiButton2"
                    />
                    <!-- Панель смайликов -->
                    <div v-if="showEmojiPanel" class="emoji-panel" ref="emojiPanel2">
                        <span v-for="emoji in emojis" :key="emoji" class="emoji" @click="insertEmoji(emoji)">
                            {{ emoji }}
                        </span>
                    </div>
                </div>
            </div>
            
             <div class="header_tools">
                <img src="@/assets/images/crop.png" @click="changeCrop">
                <img src="@/assets/images/return_back.png" @click="undo">
                <img src="@/assets/images/return_towards.png" @click="redo">
            </div>
            <div class="cropper_wrapper">
                <!-- Контейнер для vue-advanced-cropper -->
                <div v-if="imageSrc" class="cropper-container" ref="container">
                    <div
                        v-for="block in textBlocks"
                        :key="block.id"
                        class="text-block"
                        :data-id="block.id"
                        :style="{
                            top: `${block.top}px`,
                            left: `${block.left}px`,
                            fontSize: `${block.fontSize}px`,
                            color: block.color,
                            fontWeight: block.fontWeight,
                            fontStyle: block.fontStyle,
                            fontFamily: block.fontFamily,
                            textAlign: block.textAlign, // Выравнивание текста
                            cursor: 'pointer',
                            zIndex: block.zIndex,
                            display: block.display
                        }"
                        @mousedown.stop="selectBlock(block.id, $event)"
                        @touchstart.stop="selectBlock(block.id, $event)"
                    >
                        <!-- Редактируемый текст -->
                        <div
                        :contenteditable="block.id === editingBlockId"
                        @input="updateText(block.id, $event)"
                        @keydown="handleEnterDown($event)"
                        @click.stop="enableEditing(block.id); $event.stopPropagation()"
                        @blur="disableEditing(block.id)"
                        :style="{
                            whiteSpace: 'pre-wrap', // Сохраняем переносы строк
                            cursor: 'pointer',
                            maxHeight: '100%'
                        }"
                        >
                        {{ block.text }}
                        </div>
                    </div>
                    <div
                        v-for="image in images"
                        :key="image.id"
                        class="overlay-image"
                        :style="{
                        top: `${image.top}px`,
                        left: `${image.left}px`,
                        width: `${image.width}px`,
                        height: `${image.height}px`,
                        zIndex: image.zIndex,
                        transform: `rotate(${image.rotation}deg)`,
                        display: image.display,
                        }"
                        @mousedown="selectImage(image.id, $event)"
                        @touchstart="selectImage(image.id, $event)"
                    >
                    <!-- eslint-disable vue/no-use-v-if-with-v-for  -->
                        <div
                            v-if="selectedLay == image.id"
                            v-for="handle in rectangleHandles"
                            :key="handle.position"
                            :class="['handle', handle.position]"
                            @mousedown="startResizeImage($event, handle.position, image.id)"
                            @touchstart="startResizeImage($event, handle.position, image.id)"
                        ></div>
                        <div 
                            v-if="selectedLay == image.id"
                            class="rotate_handle"
                            @mousedown="startRotateImage($event, image.id)"
                            @touchstart="startRotateImage($event, image.id)"
                        >
                            <img src="@/assets/images/return_back.png" />
                        </div>
                        <img
                        :src="image.src"
                        :style="{
                            width: '100%',
                            height: '100%',
                            pointerEvents: 'none'
                        }"
                        />
                    </div>
                    <div
                        v-for="rectangle in rectangles"
                        :key="rectangle.id"
                        class="rectangle"
                        :style="{
                        top: `${rectangle.top}px`,
                        left: `${rectangle.left}px`,
                        width: `${rectangle.width}px`,
                        height: `${rectangle.height}px`,
                        backgroundColor: rectangle.color,
                        opacity: rectangle.opacity,
                        zIndex: rectangle.zIndex,
                        }"
                        @mousedown="selectRectangle(rectangle.id, $event)"
                        @touchstart="selectRectangle(rectangle.id, $event)"
                    >
                    <!-- eslint-disable vue/no-use-v-if-with-v-for  -->
                        <div
                            v-if="rectangle.id === selectedLay"
                            v-for="handle in rectangleHandles"
                            :key="handle.position"
                            :class="['handle', handle.position]"
                            @mousedown="startResizeRectangle($event, handle.position, rectangle.id)"
                            @touchstart="startResizeRectangle($event, handle.position, rectangle.id)"
                        ></div>
                    </div>
                    <cropper
                        ref="cropper"
                        class="cropper"
                        :src="currentImage"
                        :resize-image="{
                            wheel: false
                        }"
                        :stencil-props="{
                            handlers: isCroped ? 
                            {
                                eastNorth: true,
                                north: false,
                                westNorth: true,
                                west: false,
                                westSouth: true,
                                south: false,
                                eastSouth: true,
                                east: false,
                            } : {},
                        }"
                        @change="onCropChange"
                        :style="{
                            maxWidth: `${windowWidth - 30}px`,
                            background: 'none'
                        }"
                        :default-size="defaultSize"
                        @click="resetSelect"
                    />
                </div>
                <div class="crop_msg" v-if="isCroped">
                    <span>Обрезать?</span>
                    <AppGoodButton :text="text7" @click="resize" class="crop" /> 
                </div>
                <div class="row row_templates">
                    <span>Шаблон: </span>
                    <div class="row_templates_sub_row">
                        <AppGoodButton 
                            :isReboot="true"
                            class="square_btn not_active"
                            @click="rebootTemplates"
                        />
                        <AppGoodButton 
                            v-for="(item, index) in ['1', '2', '3']"
                            :text="item" 
                            :key="index"
                            class="square_btn" 
                            :class="{ not_active: index !== selectedTemplate }"
                            @click="selectTemplate(index)"
                        />
                        <AppGoodButton 
                            :isSave="true"
                            class="square_btn not_active"
                            @click="openSaveTemplate"
                        />
                    </div>
                </div>
            </div>
            <div class="text_tools" @mousedown.self="resetSelect" @touchstart.self="resetSelect">
                <div class="text_tools_row">
                    <div class="color_rectangle">
                        <input type="color" class="color_rect" v-model="selectedRectangle.color" @change="captureState" />
                        <h2>Цвет</h2>
                    </div>
                    <div class="vertical_line"></div>
                    <div class="text_text" @click="changeTextStyle('normal')">
                        <span>T</span>
                        <h2>Text</h2>
                    </div>
                    <div class="text_text" @click="changeTextStyle('bold')">
                        <span class="bold">B</span>
                        <h2>Bold</h2>
                    </div>
                    <div class="text_text">
                        <span class="italic" @click="changeTextStyle('italic')">I</span>
                        <h2>Italic</h2>
                    </div>
                    <!-- <div class="vertical_line"></div> -->
                </div>
                <div class="text_tools_row">
                    <div class="rect_opacity">
                        <div
                            class="rect_op"
                            :style="{
                                backgroundColor: `rgba(${hexToRgb(selectedRectangle.color).r}, ${hexToRgb(selectedRectangle.color).g}, ${hexToRgb(selectedRectangle.color).b}, ${selectedRectangle.opacity})`
                            }"
                        ></div>
                        <h3>Заливка: </h3>
                        <input type="range" class="custom-range custom-range-sm" v-model="selectedRectangle.opacity" min="0" max="1" step="0.01" @change="captureState" />
                    </div>
                    <div class="text_align">
                        <div class="text_align_item" @click="changeTextAling('left')">
                            <div class="text_align_left" :class="{ non_active_align: selectedBlock.textAlign != 'left' }">
                                <div></div>
                                <div></div>
                                <div></div>
                                <div></div>
                            </div>
                            <h2 :class="{ non_active_align: selectedBlock.textAlign != 'left' }">Left</h2>
                        </div>
                        <div class="text_align_item" @click="changeTextAling('center')">
                            <div class="text_align_center" :class="{ non_active_align: selectedBlock.textAlign != 'center' }">
                                <div></div>
                                <div></div>
                                <div></div>
                                <div></div>
                            </div>
                            <h2 :class="{ non_active_align: selectedBlock.textAlign != 'center' }">Center</h2>
                        </div>
                        <div class="text_align_item" @click="changeTextAling('right')">
                            <div class="text_align_right" :class="{ non_active_align: selectedBlock.textAlign != 'right' }">
                                <div></div>
                                <div></div>
                                <div></div>
                                <div></div>
                            </div>
                            <h2 :class="{ non_active_align: selectedBlock.textAlign != 'right' }">Right</h2>
                        </div>
                    </div>
                </div>
                <!-- <AppGoodButton :text="text3" class="upload_logo" @click="addImage" /> -->
                <div class="text_tools_row" style="flex-wrap: wrap;">
                    <h2>Размер текста: </h2>
                    <input type="range" class="custom-range custom-range-big" v-model.number="selectedBlock.fontSize" min="5" max="256" step="1" @change="captureState" />
                    <input class="font_size" type="number" v-model="selectedBlock.fontSize" min="5" max="256" @change="captureState" :key="selectedBlock.fontSize" />
                </div>
                <div class="text_tools_row top_align">
                    <h2>Шрифт: </h2>
                    <select class="font_dropdown" v-model="selectedBlock.fontFamily">
                        <option value="Tektur" @click="captureState">Tektur</option>
                        <option value="Montserrat" @click="captureState">Montserrat</option>
                        <option value="Arial" @click="captureState">Arial</option>
                        <option value="OpenSans" @click="captureState">Open Sans</option>
                        <option value="Roboto" @click="captureState">Roboto</option>
                    </select>
                    <div class="color_text">
                        <div 
                            class="color_circle" 
                            @click="openColorPicker" 
                            :style="{ backgroundColor: selectedBlock.color }"
                        ></div>
                        <h2>Цвет</h2>
                        <input 
                            type="color" 
                            style="visibility: hidden; height: 0px;" 
                            ref="colorInput" 
                            v-model="selectedBlock.color" 
                            @change="captureState"
                        />
                    </div>
                </div>
            </div>
            <div class="layers">
                <h2>Слои:</h2>
                <div class="line"></div>

                <draggable 
                    tag="ul"
                    :list="layers"
                    ghost-class="ghost"
                    class="layers-list"
                    @end="onDragEnd" 
                    item-key="name"
                >
                    <template #item="{element, index}">
                        <div 
                            class="layer-item"
                            :class="{ active_lay_item: selectedLay == element.id }"
                            @click="selectLay(element.id)"
                        >
                            <img src="@/assets/images/eye.png" v-if="isVisibleLay(element)" class="eye" @click="changeVisible(element)" />
                            <img src="@/assets/images/close_eye.png" v-if="!isVisibleLay(element)" class="eye" @click="changeVisible(element)" />

                            <span 
                                v-if="element.type == 'text'"
                                class="drag-handle"
                            >T</span>
                            <img 
                                v-if="element.type == 'image' && element?.link"
                                :src="element?.link"
                                class="preview"
                            />
                            <div 
                                v-if="element.type == 'rectangle'"
                                class="preview_rect"
                            ></div>
                            <span  
                                class="layer_name"    
                            >{{ element.name }}</span>
                            
                            <img 
                                class="delete_lay" 
                                src="@/assets/images/close.png" 
                                @click="deleteLayer(element, index)"
                            >
                        </div>
                    </template>
                </draggable>
                

            </div>
            <div class="row row_btns">
                <AppBadButton :text="text1" class="save" @click="save"/>
                <AppGoodButton :text="text2" class="download" @click="downloadImage"/>
            </div>
        </div>
    </div>
</template>

<script>
    import { Cropper } from 'vue-advanced-cropper';
    import draggable from 'vuedraggable';
    import 'vue-advanced-cropper/dist/style.css';
    import AppGoodButton from "@/components/AppGoodButton.vue";
    import AppBadButton from "@/components/AppBadButton.vue";
    import { getConfig } from '@/services/config';
    import { saveTemplate, getTemplates } from '@/services/ai';
    import { loadImage } from '@/services/other';

    import big_dark from '@/assets/images/big_dark.png';
    import sm_dark from '@/assets/images/sm_dark.png';
    import logo from '@/assets/images/intelektaz_logo.png';
    import rect_banner from '@/assets/images/rect_banner.png';

    export default {
        components: { AppGoodButton, AppBadButton, Cropper, draggable },
        props: {
            imageSrc: String,
            userData: Object
        },
        data() {
            return {
                text1: "СОХРАНИТЬ",
                text2: "СКАЧАТЬ",
                text3: "ЗАГРУЗИТЬ ЛОГО",
                text4: "ТЕКСТ",
                text5: "ПРЯМОУГОЛЬНИК",
                text6: "ИЗОБРАЖЕНИЕ",
                text7: "ДА",
                croppedImage: null, // Обрезанное изображение
                currentImage: null,
                fileExtension: '', // Расширение загруженного файла
                textBlocks: [], // Массив текстовых блоков
                selectedBlockId: null, // ID выбранного текстового блока
                isDraggingText: false, // Флаг для перетаскивания текста
                dragStartX: 0, // Начальная позиция мыши по X
                dragStartY: 0, // Начальная позиция мыши по Y
                cords: null,
                imageWidth: 0,
                imageHeight: 0,
                shiftX: 0,
                shiftY: 0,
                startSizeW: 0,
                startSizeH: 0,
                rectangles: [], // Массив прямоугольников
                selectedRectangle: {
                    color: '#000000',
                    opacity: 1,
                    width: 100,
                    height: 50,
                },
                selectedRectangleId: null, // ID выбранного прямоугольника
                rectangleHandles: [
                    { position: 'top-left' },
                    { position: 'top-right' },
                    { position: 'bottom-left' },
                    { position: 'bottom-right' },
                ],
                editingBlockId: null,
                images: [], // Массив добавленных изображений
                selectedImageId: null, // ID выбранного изображения
                isShiftPressed: false,
                textColor: "#000000",
                isCroped: false,
                history: [], // Стек для хранения истории действий
                future: [],  // Стек для хранения отменённых действий
                historyLimit: 50,
                canvasWidth: 0,
                resizeObservers: {},
                layers: [],
                dragging: false,
                ignoreWatch: false,
                lay_id: 1,
                active_lay: null,
                selectedLay: null,
                cursor_pos: null,
                selectedTemplate: null,
                rotationStartAngle: 0,
                isRotatingImage: false,
                initialAngle: 0,
                rotationCenterX: 0,
                rotationCenterY: 0,
                templates: [],
                emojis: [],
                showEmojiPanel: false,
                isSaveTemplate: false,
            }
        },
        mounted() {
            window.addEventListener('pointerdown', this.handleClickOutside, true);
            document.addEventListener('keydown', this.handleKeyDown);
            document.addEventListener('keyup', this.handleKeyUp);
            document.addEventListener("click", this.handleClickOutside);
            this.canvasWidth = window.innerWidth;
        },
        computed: {
            windowWidth() {
                return window.innerWidth;
            },
            selectedBlock() {
                let block = this.textBlocks.find((block) => block.id === this.selectedBlockId);
                if (block) console.log(block.top, block.fontSize);
                return block || {
                    id: Date.now(),
                    text: 'Новый текст',
                    top: 200,
                    left: 400,
                    fontSize: 20,
                    color: "#000000",
                    fontWeight: 'normal', // По умолчанию обычный текст
                    fontStyle: 'normal',  // По умолчанию без курсива
                    fontFamily: 'Arial',  // По умолчанию Arial
                    width: 200,           // Ширина текстового блока
                    textAlign: 'left',
                };
            },
        },
        async created() {
            this.currentImage = this.imageSrc;

            await this.waitForImageLoad();
            await this.waitForPreviewReady();

            this.getShift();
            const wrapper = document.querySelector('.vue-preview__wrapper');
            const background = document.querySelector('.vue-advanced-cropper__foreground');
            const background2 = document.querySelector('.vue-advanced-cropper__background');

            if (wrapper) {
                this.startSizeW = wrapper.offsetWidth;
                this.startSizeH = wrapper.offsetHeight;
            }

            if (background) background.style.background = 'none';
            if (background2) background2.style.background = 'none';

            // Загружаем шаблоны, когда всё готово
            this.setTemplates();
            const temps = await getTemplates(this.userData.id);
            temps.templates.forEach((item, index) => {
                if (item.length > 0) this.templates[index] = JSON.parse(item);
            });

            // Эмодзи
            const resp = await getConfig('emojis', localStorage.getItem('token'));
            this.emojis = resp.emojis;
        },
        methods: {
                waitForImageLoad() {
                    return new Promise((resolve) => {
                        const imageEl = document.querySelector('.vue-advanced-cropper__image, .vue-preview__image');

                        if (imageEl && imageEl.complete) {
                            resolve();
                        } else if (imageEl) {
                            imageEl.addEventListener('load', () => resolve(), { once: true });
                        } else {
                            // если элемент еще не в DOM — ждём появления
                            const observer = new MutationObserver(() => {
                                const img = document.querySelector('.vue-advanced-cropper__image, .vue-preview__image');
                                if (img && img.complete) {
                                    observer.disconnect();
                                    resolve();
                                } else if (img) {
                                    img.addEventListener('load', () => {
                                        observer.disconnect();
                                        resolve();
                                    }, { once: true });
                                }
                            });

                            observer.observe(document.body, { childList: true, subtree: true });
                        }
                    });
                },

                waitForPreviewReady() {
                    return new Promise((resolve) => {
                        const check = () => {
                            const wrapper = document.querySelector('.vue-preview__wrapper');
                            const bg1 = document.querySelector('.vue-advanced-cropper__foreground');
                            const bg2 = document.querySelector('.vue-advanced-cropper__background');

                            if (wrapper && wrapper.offsetWidth > 0 && wrapper.offsetHeight > 0 && bg1 && bg2) {
                                resolve();
                            } else {
                                requestAnimationFrame(check);
                            }
                        };
                        requestAnimationFrame(check);
                    });
                },

            async saveTemplate(index) {
                if (index >= this.templates.length) return;

                this.captureState();
                let state = this.history[this.history.length - 1];
                state.selectedTemplate = index;

                this.templates[index] = state;
                this.selectedTemplate = index;

                const resp = await saveTemplate(this.userData.id, index, JSON.stringify(state));
                console.log(resp);

                this.closeSaveTemplate();
            },  
            closeSaveTemplate() {
                this.isSaveTemplate = false;
            },
            rebootTemplates() {
                const state_reboot = {
                    textBlocks: JSON.parse(JSON.stringify([])),
                    rectangles: JSON.parse(JSON.stringify([])),
                    images: JSON.parse(JSON.stringify([])),
                    layers: JSON.parse(JSON.stringify([])),
                    selectedLay: null,
                    selectedTemplate: null
                };
                // this.addHistory(state_reboot);
                this.restoreState(state_reboot);
            },
            openSaveTemplate() {
                this.isSaveTemplate = true;
            },
            handleClickOutside(event) {
                console.log('click');
                const isInside = event.target.closest('.text-block');
                if (!isInside) {
                    this.editingBlockId = null;
                }

                const panelRef = this.windowWidth > 900 ? this.$refs.emojiPanel : this.$refs.emojiPanel2;
                const buttonRef = this.windowWidth > 900 ? this.$refs.emojiButton : this.$refs.emojiButton2;

                const panel = panelRef?.$el || panelRef;
                const button = buttonRef?.$el || buttonRef;

                if (
                    panel &&
                    !panel.contains(event.target) &&
                    button &&
                    !button.contains(event.target)
                ) {
                    this.showEmojiPanel = false;
                }
            },
            insertEmoji(emoji) {
                const block = this.textBlocks.find(t => t.id == this.selectedLay);
                if (block.length == 0) return;

                block.text += emoji;
            },
            toggleEmojiPanel() {
                this.showEmojiPanel = !this.showEmojiPanel;
            },
            setTemplates() {
                // инициализация данных
                let imagesTemplate = [];
                let rectanglesTemplate = [];
                let textBlocksTemplate = [];
                let layersTemplate = [];
                let selectedLayTemplate = null;
                let selectedTemplateTemplate = 0;
                // let croppedImageTemplate = 'https://api.intelektaz.com/assets/b141af93-948f-4179-918b-e67dc22d6ee9';
                
                // размеры холста
                const bounds = this.getCropperBounds();
                console.log(bounds);

                this.cropperWidth = bounds.width;

                // масштаб
                const scaleX = bounds.width / 1280;
                const scaleY = bounds.height / 1280;

                // затемнение снизу
                let newImage = {
                    id: 1,
                    top: bounds.top + 718 * scaleY,
                    left: bounds.left + (-308) * scaleX,
                    width: bounds.width * 1.4359375,
                    height: bounds.height * 0.596875,
                    src: sm_dark,
                    zIndex: 1,
                    rotation: 0,
                    display: 'block',
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Затеменение снизу`, id: newImage.id, type: "image", link: newImage.src });

                // затемнение сверху
                newImage = {
                    id: 2,
                    top: bounds.top + (-50) * scaleY,
                    left: bounds.left + (-164) * scaleX,
                    width: bounds.width * 1.2390625,
                    height: bounds.height * 0.2140625,
                    src: big_dark,
                    zIndex: 1,
                    rotation: 0,
                    display: 'block',
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Затеменение сверху`, id: newImage.id, type: "image", link: newImage.src });

                

                // логотип
                newImage = {
                    id: 3,
                    top: bounds.top + 53 * scaleY,
                    left: bounds.left + 436 * scaleX,
                    width: bounds.width * 0.0765625,
                    height: bounds.width * 0.0765625,
                    src: logo,
                    zIndex: 2,
                    rotation: 0,
                    display: 'block'
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Логотип ${this.images.length}`, id: newImage.id, type: "image", link: newImage.src })
                
                // красный фон для текста
                let newRectangle = {
                    id: 4,
                    top: bounds.top + 950 * scaleY,
                    left: bounds.left,
                    width: bounds.width,
                    height: 260 * scaleY,
                    color: '#DE5386',
                    opacity: 1,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                rectanglesTemplate.push(newRectangle);
                layersTemplate.unshift({ name: `Прямоугольник ${rectanglesTemplate.length}`, id: newRectangle.id, type: "rectangle" });

                // текст к логотипу
                let newBlock = {
                    id: 5,
                    text: 'INTELEKTAZ',
                    top: bounds.top + 73 * scaleY,
                    left: bounds.left + 553 * scaleX,
                    fontSize: 48 * scaleY,
                    color: 'white',
                    fontWeight: 'normal',
                    fontStyle: 'normal',
                    fontFamily: 'Arial',
                    textAlign: 'left',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                // текст белым снизу
                newBlock = {
                    id: 6,
                    text: 'Перестань выживать — начни доминировать!',
                    top: bounds.top + 839 * scaleY,
                    left: bounds.left + 74 * scaleX,
                    fontSize: 48 * scaleY,
                    color: 'white',
                    fontWeight: 'normal',
                    fontStyle: 'normal',
                    fontFamily: 'OpenSans',
                    textAlign: 'left',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                
                // большой текст на красном фоне
                newBlock = {
                    id: 7,
                    text: 'ИИ ПРЕВРАЩАЕТ СОЦСЕТИ \nВ ЗОЛОТУЮ ЖИЛУ!',
                    top: bounds.top + 960 * scaleY,
                    left: bounds.left + 51 * scaleX,
                    fontSize: 90 * scaleY,
                    color: 'white',
                    fontWeight: 'bold',
                    fontStyle: 'normal',
                    fontFamily: 'Tektur',
                    textAlign: 'center',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });
                

                selectedLayTemplate = newImage.id;

                // итоговый кадр
                const state = {
                    textBlocks: JSON.parse(JSON.stringify(textBlocksTemplate)),
                    rectangles: JSON.parse(JSON.stringify(rectanglesTemplate)),
                    images: JSON.parse(JSON.stringify(imagesTemplate)),
                    layers: JSON.parse(JSON.stringify(layersTemplate)),
                    selectedLay: selectedLayTemplate,
                    selectedTemplate: selectedTemplateTemplate
                };
                this.templates.push(state);


                // Шаблон 2
                imagesTemplate = [];
                rectanglesTemplate = [];
                textBlocksTemplate = [];
                layersTemplate = [];
                selectedTemplateTemplate = 1;
                // croppedImageTemplate = 'https://api.intelektaz.com/assets/fc4846e7-46e5-4360-97ae-153c7464f225';
                
                // затемнение снизу
                newImage = {
                    id: 8,
                    top: bounds.top + 718 * scaleY,
                    left: bounds.left + (-308) * scaleX,
                    width: bounds.width * 1.4359375,
                    height: bounds.height * 0.596875,
                    src: sm_dark,
                    zIndex: 1,
                    rotation: 0,
                    display: 'block',
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Затеменение снизу`, id: newImage.id, type: "image", link: newImage.src });

                // затемнение сверху
                newImage = {
                    id: 9,
                    top: bounds.top + (-50) * scaleY,
                    left: bounds.left + (-164) * scaleX,
                    width: bounds.width * 1.2390625,
                    height: bounds.height * 0.2140625,
                    src: big_dark,
                    zIndex: 1,
                    rotation: 0,
                    display: 'block',
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Затеменение сверху`, id: newImage.id, type: "image", link: newImage.src });

                

                // логотип
                newImage = {
                    id: 10,
                    top: bounds.top + 53 * scaleY,
                    left: bounds.left + 436 * scaleX,
                    width: bounds.width * 0.0765625,
                    height: bounds.width * 0.0765625,
                    src: logo,
                    zIndex: 2,
                    rotation: 0,
                    display: 'block'
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Логотип ${this.images.length}`, id: newImage.id, type: "image", link: newImage.src })

                // текст к логотипу
                newBlock = {
                    id: 11,
                    text: 'INTELEKTAZ',
                    top: bounds.top + 73 * scaleY,
                    left: bounds.left + 553 * scaleX,
                    fontSize: 48 * scaleY,
                    color: 'white',
                    fontWeight: 'normal',
                    fontStyle: 'normal',
                    fontFamily: 'Arial',
                    textAlign: 'left',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                // текст белым снизу
                newBlock = {
                    id: 12,
                    text: 'ТВОЙ БИЗНЕС ЗАХЛЕБЫВАЕТСЯ',
                    top: bounds.top + 802 * scaleY,
                    left: bounds.left + 60 * scaleX,
                    fontSize: 75.75 * scaleY,
                    color: 'white',
                    fontWeight: 'bold',
                    fontStyle: 'normal',
                    fontFamily: 'Tektur',
                    textAlign: 'left',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                
                // большой текст на красном фоне
                newBlock = {
                    id: 13,
                    text: 'В РУТИНЕ?',
                    top: bounds.top + 870 * scaleY,
                    left: bounds.left + 60 * scaleX,
                    fontSize: 222.87 * scaleY,
                    color: '#BAFF26',
                    fontWeight: 'bold',
                    fontStyle: 'normal',
                    fontFamily: 'Tektur',
                    textAlign: 'center',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                // большой текст на красном фоне
                newBlock = {
                    id: 14,
                    text: 'Выпусти на свободу ИИ и получи взрывной рост продаж!',
                    top: bounds.top + 1136 * scaleY,
                    left: bounds.left + 98 * scaleX,
                    fontSize: 36 * scaleY,
                    color: 'white',
                    fontWeight: 'normal',
                    fontStyle: 'normal',
                    fontFamily: 'OpenSans',
                    textAlign: 'center',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });
                

                selectedLayTemplate = newImage.id;

                // итоговый кадр
                const state2 = {
                    textBlocks: JSON.parse(JSON.stringify(textBlocksTemplate)),
                    rectangles: JSON.parse(JSON.stringify(rectanglesTemplate)),
                    images: JSON.parse(JSON.stringify(imagesTemplate)),
                    layers: JSON.parse(JSON.stringify(layersTemplate)),
                    selectedLay: selectedLayTemplate,
                    selectedTemplate: selectedTemplateTemplate
                };
                this.templates.push(state2);


                // Шаблон 3
                imagesTemplate = [];
                rectanglesTemplate = [];
                textBlocksTemplate = [];
                layersTemplate = [];
                selectedTemplateTemplate = 2;
                // croppedImageTemplate = 'https://api.intelektaz.com/assets/b141af93-948f-4179-918b-e67dc22d6ee9';
                
                // затемнение снизу
                newImage = {
                    id: 15,
                    top: bounds.top + 718 * scaleY,
                    left: bounds.left + (-308) * scaleX,
                    width: bounds.width * 1.4359375,
                    height: bounds.height * 0.596875,
                    src: sm_dark,
                    zIndex: 1,
                    rotation: 0,
                    display: 'block',
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Затеменение снизу`, id: newImage.id, type: "image", link: newImage.src });
                

                // логотип
                newImage = {
                    id: 16,
                    top: bounds.top + 53 * scaleY,
                    left: bounds.left + 436 * scaleX,
                    width: bounds.width * 0.0765625,
                    height: bounds.width * 0.0765625,
                    src: logo,
                    zIndex: 2,
                    rotation: 0,
                    display: 'block'
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Логотип ${this.images.length}`, id: newImage.id, type: "image", link: newImage.src })

                // логотип
                newImage = {
                    id: 17,
                    top: bounds.top + 1090 * scaleY,
                    left: bounds.left + 99 * scaleX,
                    width: bounds.width * 0.84453125,
                    height: bounds.height * 0.09375,
                    src: rect_banner,
                    zIndex: 2,
                    rotation: 0,
                    display: 'block'
                };
                imagesTemplate.push(newImage);
                layersTemplate.unshift({ name: `Фон под текст`, id: newImage.id, type: "image", link: newImage.src })

                // текст к логотипу
                newBlock = {
                    id: 18,
                    text: 'INTELEKTAZ',
                    top: bounds.top + 73 * scaleY,
                    left: bounds.left + 553 * scaleX,
                    fontSize: 48 * scaleY,
                    color: 'white',
                    fontWeight: 'normal',
                    fontStyle: 'normal',
                    fontFamily: 'Arial',
                    textAlign: 'left',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                // текст красным
                newBlock = {
                    id: 19,
                    text: 'СЕКРЕТНЫЙ',
                    top: bounds.top + 176 * scaleY,
                    left: bounds.left + 141 * scaleX,
                    fontSize: 171.53 * scaleY,
                    color: '#9C1A1D',
                    fontWeight: 'bold',
                    fontStyle: 'normal',
                    fontFamily: 'Tektur',
                    textAlign: 'left',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                
                // большой текст на красном фоне
                newBlock = {
                    id: 20,
                    text: 'ИНГРЕДИЕНТ УСПЕХА:',
                    top: bounds.top + 376 * scaleY,
                    left: bounds.left + 141 * scaleX,
                    fontSize: 94.99 * scaleY,
                    color: 'white',
                    fontWeight: 'bold',
                    fontStyle: 'normal',
                    fontFamily: 'Tektur',
                    textAlign: 'center',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });

                // большой текст на красном фоне
                newBlock = {
                    id: 21,
                    text: 'ИИ, который привлекает клиентов 24/7',
                    top: bounds.top + 1121 * scaleY,
                    left: bounds.left + 139 * scaleX + 10,
                    fontSize: 48 * scaleY,
                    color: 'white',
                    fontWeight: 'normal',
                    fontStyle: 'normal',
                    fontFamily: 'OpenSans',
                    textAlign: 'center',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + layersTemplate.length,
                    display: 'block'
                };
                textBlocksTemplate.push(newBlock);
                layersTemplate.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });
                

                selectedLayTemplate = newImage.id;

                // итоговый кадр
                const state3 = {
                    textBlocks: JSON.parse(JSON.stringify(textBlocksTemplate)),
                    rectangles: JSON.parse(JSON.stringify(rectanglesTemplate)),
                    images: JSON.parse(JSON.stringify(imagesTemplate)),
                    layers: JSON.parse(JSON.stringify(layersTemplate)),
                    selectedLay: selectedLayTemplate,
                    selectedTemplate: selectedTemplateTemplate
                };
                this.templates.push(state3);
            },
            activateTemplate(index) {
                if (index >= this.templates.length) return;
                const template = this.templates[index];
                this.restoreState(template);
            },
            isVisibleLay(layer) {
                if (layer.type === 'text') {
                    const item = this.textBlocks.find(item => item.id === layer.id);
                    return item ? item.display === 'block' : false;
                } else if (layer.type === 'image') {
                    const item = this.images.find(item => item.id === layer.id);
                    return item ? item.display === 'block' : false;
                } else {
                    const item = this.rectangles.find(item => item.id === layer.id);
                    return item ? item.display === 'block' : false;
                }
            },
  
            changeVisible(layer) {
                if (layer.type === 'text') {
                    this.textBlocks.forEach(item => {
                        if (item.id == layer.id) {
                            item.display = item.display == 'block' ? 'none' : 'block';
                        }
                    });
                } else if (layer.type === 'image') {
                    this.images.forEach(item => {
                        if (item.id == layer.id) {
                            item.display = item.display == 'block' ? 'none' : 'block';
                        }
                    });
                } else if (layer.type === 'rectangle') {
                    this.rectangles.forEach(item => {
                        if (item.id == layer.id) {
                            item.display = item.display == 'block' ? 'none' : 'block';
                        }
                    });
                }
            },
            startRotateImage(event, id) {
                const image = this.images.find(img => img.id === id);
                if (!image) return;

                this.isRotatingImage = true;
                this.selectedImageId = id;

                this.initialAngle = image.rotation || 0;

                const imageElement = event.currentTarget.closest('.overlay-image');
                const rect = imageElement.getBoundingClientRect();
                const centerX = rect.left + rect.width / 2;
                const centerY = rect.top + rect.height / 2;

                this.rotationCenterX = centerX;
                this.rotationCenterY = centerY;

                const startX = event.touches ? event.touches[0].pageX : event.pageX;
                const startY = event.touches ? event.touches[0].pageY : event.pageY;

                this.rotationStartAngle = Math.atan2(startY - centerY, startX - centerX) * (180 / Math.PI) - this.initialAngle;

                document.addEventListener('mousemove', this.rotateImage);
                document.addEventListener('touchmove', this.rotateImage, { passive: false });
                document.addEventListener('mouseup', this.stopRotateImage);
                document.addEventListener('touchend', this.stopRotateImage);

                event.preventDefault();
            },
            
            rotateImage(event) {
                if (!this.isRotatingImage || !this.selectedImageId) return;

                const image = this.images.find(img => img.id === this.selectedImageId);
                if (!image) return;

                const currentX = event.touches ? event.touches[0].pageX : event.pageX;
                const currentY = event.touches ? event.touches[0].pageY : event.pageY;

                const currentAngle = Math.atan2(currentY - this.rotationCenterY, currentX - this.rotationCenterX) * (180 / Math.PI);

                image.rotation = currentAngle - this.rotationStartAngle;
                image.rotation = (image.rotation % 360 + 360) % 360; // нормализация
            },

            
            stopRotateImage() {
                this.isRotatingImage = false;
                document.removeEventListener('mousemove', this.rotateImage);
                document.removeEventListener('touchmove', this.rotateImage);
                document.removeEventListener('mouseup', this.stopRotateImage);
                document.removeEventListener('touchend', this.stopRotateImage);
            },
            selectLay(id) {
                this.selectedLay = id;
            },
            selectTemplate(index) {
                this.selectedTemplate = this.selectedTemplate == index ? null : index;
                this.activateTemplate(index);
            },
            resetSelect() {
                this.selectedLay = null;
                console.log('resetSelect');
            },
            isOnlyZIndexChanged(newArray, oldArray) {
                console.log(newArray, oldArray);
                for (let i = 0; i < newArray.length; i++) {
                    const newItem = { ...newArray[i] };
                    const oldItem = { ...oldArray[i] };

                    console.log(newItem, oldItem);

                    if (newItem.zIndex == oldItem.zIndex) return false;

                    // Удаляем свойство zIndex из сравнения
                    delete newItem.zIndex;
                    delete oldItem.zIndex;

                    // Сравниваем оставшиеся свойства
                    console.log(JSON.stringify(newItem) !== JSON.stringify(oldItem), JSON.stringify(newItem) != JSON.stringify(oldItem), JSON.stringify(newItem), JSON.stringify(oldItem));
                    if (JSON.stringify(newItem) !== JSON.stringify(oldItem)) {
                        return false;
                    }
                }

                return true;
            },
            onDragEnd() {
                this.ignoreWatch = true;
                this.layers
                    .slice() // Создаем копию массива, чтобы не мутировать оригинальный массив
                    .reverse() // Переворачиваем массив для обхода с конца
                    .forEach((layer, reverseIndex) => {
                        const zIndex = reverseIndex + 1; // Начинаем zIndex с 1

                        if (layer.type === 'text') {
                            const block = this.textBlocks.find(b => b.id === layer.id);
                            if (block) block.zIndex = zIndex;
                        } else if (layer.type === 'image') {
                            const image = this.images.find(img => img.id === layer.id);
                            if (image) image.zIndex = zIndex;
                        } else if (layer.type === 'rectangle') {
                            const rect = this.rectangles.find(r => r.id === layer.id);
                            if (rect) rect.zIndex = zIndex;
                        }
                    });
                this.ignoreWatch = false;
            },
            deleteLayer(layer, index) {
                console.log(layer, index);
                if (layer.type === 'text') {
                    console.log("удаление текста");
                    this.deleteBlock(layer.id);
                } else if (layer.type === 'image') {
                    console.log("удаление картинки");
                    this.deleteImage(layer.id);
                } else if (layer.type === 'rectangle') {
                    console.log("удаление прямоугольника");
                    this.deleteRectangle(layer.id);
                }
            },
            initResizeObserver(id) {
                const blockElement = document.querySelector(`.text-block[data-id="${id}"]`);
                if (!blockElement) return;

                // Создаем нового наблюдателя
                this.resizeObservers[id] = new ResizeObserver(entries => {
                for (let entry of entries) {
                    const { width, height } = entry.contentRect;
                    const block = this.textBlocks.find(b => b.id === id);
                    if (block) {
                    block.width = Math.round(width);
                    block.height = Math.round(height);
                    }
                }
                });

                // Начинаем наблюдение
                this.resizeObservers[id].observe(blockElement);
            },
            getCropperBounds() {
                const cropperImage = document.querySelector('.vue-advanced-cropper__image');
                const cropperImageWrapper = document.querySelector('.vue-advanced-cropper__image-wrapper');
                
                if (!cropperImage) return null;

                const rect = cropperImage.getBoundingClientRect();

                let left = 0;
                let top = 0;

                left = (cropperImageWrapper.offsetWidth - rect.width) / 2;

                return {
                    left: left,
                    top: top,
                    width: rect.width,
                    height: rect.height,
                    right: left,
                    bottom: top
                };
            },
            // Метод для добавления действия в историю
            addHistory(state) {
                this.history.push(state);
                if (this.history.length > this.historyLimit) {
                    this.history.shift(); // Удаляем самое старое действие при превышении лимита
                }
                this.future = []; // При новом действии очищаем стек "вперёд"
            },
            // Метод для создания снимка текущего состояния
            captureState() {
                const state = {
                    textBlocks: JSON.parse(JSON.stringify(this.textBlocks)),
                    rectangles: JSON.parse(JSON.stringify(this.rectangles)),
                    images: JSON.parse(JSON.stringify(this.images)),
                    layers: JSON.parse(JSON.stringify(this.layers)),
                    selectedLay: this.selectedLay,
                    selectedTemplate: this.selectedTemplate
                };
                this.addHistory(state);
            },
            undo() {
                if (this.history.length == 1) return; // Нечего отменять
                
                const currentState = this.history.pop(); // Текущее состояние
                this.future.push(currentState); // Перемещаем текущее состояние в будущее
                
                const prevState = this.history[this.history.length - 1];
                this.restoreState(prevState);

                console.log(this.history);
            },
            
            redo() {
                console.log(this.future);
                if (this.future.length === 0) return; // Нечего возвращать
                
                const nextState = this.future.pop();
                // this.addHistory(nextState); // Добавляем восстановленное состояние в историю
                this.restoreState(nextState);
            },
            restoreState(state) {
                console.log(state);
                this.textBlocks = JSON.parse(JSON.stringify(state.textBlocks));
                this.rectangles = JSON.parse(JSON.stringify(state.rectangles));
                this.images = JSON.parse(JSON.stringify(state.images));
                this.layers = JSON.parse(JSON.stringify(state.layers));
                this.selectedLay = state.selectedLay;
                this.selectedTemplate = state.selectedTemplate;

                this.$nextTick(() => {
                    for (const block of this.textBlocks) {
                        this.initResizeObserver(block.id);
                    }
                })

                console.log(this.images);
            },
            close() {
                this.$emit('update:visibility1', false);
            },
            save() {
                this.crop();
                
                setTimeout(() => {
                    this.$emit('update:save', this.croppedImage);
                    this.close();
                }, 300)
 
                
            },
            changeCrop() {
                this.isCroped = !this.isCroped;
            },
            hexToRgb(hex) {
                // Удаляем символ #, если он есть
                hex = hex.replace(/^#/, '');
                // Разбиваем шестнадцатеричное значение на компоненты RGB
                const bigint = parseInt(hex, 16);
                const r = (bigint >> 16) & 255;
                const g = (bigint >> 8) & 255;
                const b = bigint & 255;
                return { r, g, b };
            },
            changeTextAling(align) {
                this.selectedBlock.textAlign = align;
                if (this.selectedLay) {
                    const layer = this.layers.find(item => item.id == this.selectedLay);
                    const bounds = this.getCropperBounds();
                    if (layer.type === 'text') {
                        this.textBlocks.forEach(item => {
                            if (item.id === layer.id) {
                                switch (align) {
                                    case 'left':
                                        item.left = 0;
                                        break;
                                    case 'center':
                                        item.left = bounds.width / 2 - item.width / 2;
                                        break;
                                    case 'right':
                                        item.left = bounds.width - item.width - 2;
                                        break;
                                }
                            }
                        });
                    } else if (layer.type == 'image') {
                        this.images.forEach(item => {
                            if (item.id === layer.id) {
                                switch (align) {
                                    case 'left':
                                        item.left = 0;
                                        break;
                                    case 'center':
                                        item.left = bounds.width / 2 - item.width / 2;
                                        break;
                                    case 'right':
                                        item.left = bounds.width - item.width;
                                        break;
                                }
                            }
                        });
                    } else {
                        this.rectangles.forEach(item => {
                            if (item.id === layer.id) {
                                switch (align) {
                                    case 'left':
                                        item.left = 0;
                                        break;
                                    case 'center':
                                        item.left = bounds.width / 2 - item.width / 2;
                                        break;
                                    case 'right':
                                        item.left = bounds.width - item.width;
                                        break;
                                }
                            }
                        });
                    }
                }
                this.captureState();
            },
            changeTextStyle(style) {
                switch (style) {
                    case 'normal':
                        this.selectedBlock.fontWeight = style;
                        break;
                    case 'bold':
                        this.selectedBlock.fontWeight = this.selectedBlock.fontWeight == 'bold' ? 'normal' : style;
                        break;
                    case 'italic':
                        this.selectedBlock.fontStyle = this.selectedBlock.fontStyle == 'italic' ? 'normal' : style;
                        break;
                }
                this.captureState();
            },
            openColorPicker() {
                // Имитация клика по скрытому input[type="color"]
                this.$refs.colorInput.click();
            },
            handleKeyDown(event) {
                if (event.key === 'Shift') {
                    this.isShiftPressed = true;
                }
            },
            handleKeyUp(event) {
                if (event.key === 'Shift') {
                    this.isShiftPressed = false;
                }
            },
            selectImage(id, event) {
                if (!this.isResizingImage && !this.isRotatingImage) {
                    this.selectedImageId = id;
                    this.selectedLay = id;
                    this.startDragImage(event, id);
                    event.preventDefault();
                }
            },
            startDragImage(event, id) {
                const image = this.images.find((img) => img.id === id);
                if (!image) return;

                // Определяем координаты события (мыши или касания)
                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                this.isDraggingImage = true;
                this.dragStartX = clientX - image.left;
                this.dragStartY = clientY - image.top;

                this._touchMoveImageHandler = this.dragImage.bind(this);
            
                // Добавляем обработчики для мыши и касаний
                document.addEventListener('mousemove', this.dragImage);
                document.addEventListener('touchmove', this._touchMoveImageHandler, { passive: false });
                document.addEventListener('mouseup', this.stopDragImage);
                document.addEventListener('touchend', this.stopDragImage, false);
                event.preventDefault();
            },
            dragImage(event) {
                event.preventDefault();
                if (!(event.touches && event.target.classList.contains('overlay-image')) && (!this.isDraggingImage || !this.selectedImageId)) return;

                const image = this.images.find((img) => img.id === this.selectedImageId);
                if (!image) return;

                const bounds = this.getCropperBounds();
                if (!bounds) return;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                let newLeft = clientX - this.dragStartX;
                let newTop = clientY - this.dragStartY;
                

                // Ограничение по горизонтали
                // if (newLeft < bounds.left) {
                //     console.log('за границей!');
                //     newLeft = bounds.left;
                // } 
                // if (newLeft + image.width > bounds.width + bounds.left) newLeft = bounds.width + bounds.left - image.width;
                
                // // Ограничение по вертикали
                // if (newTop < bounds.top) newTop = bounds.top;
                // if (newTop + image.height > bounds.height) newTop = bounds.height - image.height;

                // if (newLeft < 0) newLeft = 0;
                // if (newTop < 0) newTop = 0;
                
                image.left = newLeft;
                image.top = newTop;
            },
            stopDragImage() {
                console.log("stop");
                this.isDraggingImage = false;

                // Удаляем обработчики для мыши и касаний
                document.removeEventListener('mousemove', this.dragImage);
                if (this._touchMoveImageHandler) {
                    document.removeEventListener('touchmove', this._touchMoveImageHandler);
                    this._touchMoveImageHandler = null;
                }
                document.removeEventListener('mouseup', this.stopDragImage);
                document.removeEventListener('touchend', this.stopDragImage);
            },
            startResizeImage(event, position, id) {
                this.isResizingImage = true;
                this.resizePosition = position;

                const image = this.images.find((img) => img.id === id);
                if (!image) return;

                // Сохраняем начальные значения
                this.initialImageLeft = image.left;
                this.initialImageTop = image.top;
                this.initialImageWidth = image.width;
                this.initialImageHeight = image.height;

                // Сохраняем начальные координаты мыши
                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                this.startX = clientX;
                this.startY = clientY;

                document.addEventListener('mousemove', this.resizeImage);
                document.addEventListener('touchmove', (event) => {
                    this.resizeImage(event);
                }, { passive: false });
                document.addEventListener('mouseup', this.stopResizeImage);
                document.addEventListener('touchend', this.stopResizeImage, false);
                event.preventDefault();
            },
            resizeImage(event) {
                if (!(event.touches && event.target.classList.contains('handles')) && (!this.isResizingImage || !this.selectedImageId)) return;

                const image = this.images.find((img) => img.id === this.selectedImageId);
                if (!image) return;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                const bounds = this.getCropperBounds();
                if (!bounds) return;

                const deltaX = clientX - this.startX;
                const deltaY = clientY - this.startY;

                // Сохраняем исходное соотношение сторон
                const aspectRatio = this.initialImageWidth / this.initialImageHeight;

                switch (this.resizePosition) {
                    case 'top-left':
                        // if (this.initialImageLeft + deltaX >= bounds.left && this.initialImageTop + (this.initialImageHeight - image.height) >= bounds.top) {
                            image.width = this.initialImageWidth - deltaX;
                            image.height = image.width / aspectRatio; // Пропорциональная высота
                            image.left = this.initialImageLeft + deltaX;
                            image.top = this.initialImageTop + (this.initialImageHeight - image.height);
                        // }
                        break;

                    case 'top-right':
                        // if (this.initialImageTop + (this.initialImageHeight - image.height) >= bounds.top && this.initialImageLeft + this.initialImageWidth + deltaX <= bounds.left + bounds.width) {
                            image.width = this.initialImageWidth + deltaX;
                            image.height = image.width / aspectRatio; // Пропорциональная высота
                            image.top = this.initialImageTop + (this.initialImageHeight - image.height);
                        // }
                        break;

                    case 'bottom-left':
                        // if (this.initialImageLeft - (image.width - this.initialImageWidth) >= bounds.left && this.initialImageTop + this.initialImageHeight + deltaY <= bounds.height) {
                            image.height = this.initialImageHeight + deltaY;
                            image.width = image.height * aspectRatio; // Пропорциональная ширина
                            image.left = this.initialImageLeft - (image.width - this.initialImageWidth);
                        // }
                        break;

                    case 'bottom-right':
                        // if (this.initialImageTop + this.initialImageHeight + deltaY <= bounds.height && this.initialImageLeft + this.initialImageWidth + deltaX <= bounds.left + bounds.width) {
                            image.width = this.initialImageWidth + deltaX;
                            image.height = image.width / aspectRatio; // Пропорциональная высота
                        // }
                        break;
                }

                // Ограничение минимальных размеров
                if (image.width < 10) {
                    image.width = 10;
                    image.height = image.width / aspectRatio; // Пропорциональная высота
                    if (this.resizePosition.includes('left')) {
                        image.left -= 10 - image.width; // Корректируем позицию
                    }
                }

                if (image.height < 10) {
                    image.height = 10;
                    image.width = image.height * aspectRatio; // Пропорциональная ширина
                    if (this.resizePosition.includes('top')) {
                        image.top -= 10 - image.height; // Корректируем позицию
                    }
                }

                // Ограничение перемещения за пределы контейнера
                if (image.left < 0) image.left = 0;
                if (image.top < 0) image.top = 0;
            },
            stopResizeImage() {
                this.isResizingImage = false;
                this.captureState();
                document.removeEventListener('mousemove', this.resizeImage);
                document.removeEventListener('mouseup', this.stopResizeImage);
                document.addEventListener('touchmove', this.resizeImage);
                document.addEventListener('touchend', this.stopResizeImage);
            },
            async addImage() {
                const input = document.createElement('input');
                input.type = 'file';
                input.accept = 'image/*';
                input.onchange = async (event) => {
                    const file = event.target.files[0];
                    if (!file) return;
                    
                    const link = await loadImage(file);

                    const reader = new FileReader();
                    reader.onload = () => {
                        const img = new Image(); // Создаем объект Image для загрузки изображения
                        img.src = link.image_id; // Устанавливаем источник изображения

                        img.onload = () => {
                            // Вычисляем пропорции изображения
                            const originalWidth = img.width;
                            const originalHeight = img.height;
                            const aspectRatio = originalHeight / originalWidth;

                            // Устанавливаем ширину 200 и вычисляем высоту с сохранением пропорций
                            const width = 200;
                            const height = width * aspectRatio;

                            const bounds = this.getCropperBounds();

                            // Создаем новый объект изображения
                            const newImage = {
                                id: Date.now(),
                                top: bounds.top + bounds.height / 2 - height / 2,
                                left: bounds.left + bounds.width / 2 - width / 2,
                                width: width,
                                height: height,
                                src: img.src,
                                zIndex: 1 + this.layers.length,
                                rotation: 0,
                                display: 'block'
                            };

                            // Добавляем изображение в массив и выбираем его
                            this.images.push(newImage);
                            this.layers.unshift({ name: `Изображение ${this.images.length}`, id: newImage.id, type: "image", link: newImage.src });
                            this.selectedImageId = newImage.id;
                            this.selectedLay = newImage.id;
                        };
                    };
                    reader.readAsDataURL(file);
                };
                input.click();
            },
            deleteImage(id) {
                console.log(id);
                this.images = this.images.filter((img) => img.id !== id);
                this.layers = this.layers.filter((img) => img.id !== id);
                if (this.selectedImageId === id) {
                    this.selectedImageId = null;
                }
                this.captureState();
            },
            enableEditing(id) {
                this.editingBlockId = id;
                this.$nextTick(() => {
                    const editableElement = document.querySelector(`.text-block[data-id="${id}"]`);
                    console.log(editableElement);
                    if (editableElement) {
                        editableElement.focus();
                    }
                });
            },
            disableEditing() {
                console.log('disable');
                this.editingBlockId = null;
            },
            updateText(id, event) {
                const block = this.textBlocks.find((b) => b.id === id);
                if (!block) return;

                const bounds = this.getCropperBounds();
                const bottom = bounds.top + bounds.height;
                console.log('enter');
                if (block.width + block.left + 30 > bounds.width + bounds.left) {
                    block.left = 0;
                    console.log('right border');
                }
                if (block.top + block.height + block.fontSize * 2 > bottom) {
                    block.top = bounds.top + bounds.height - block.height;
                    console.log('enter border');
                }
                

                // Получаем текущее значение из contenteditable элемента
                const newText = event.target.innerText;

                // Убедитесь, что текст обновляется корректно
                if (block.text !== newText) {
                    block.text = newText; // Обновляем текст из содержимого элемента
                    this.layers.forEach(item => {
                        if (item.id === id) item.name = newText;
                    })
                    this.captureState(); // Сохраняем состояние после изменения
                }

                // Сохраняем текущую позицию курсора
                const selection = window.getSelection();
                if (selection.rangeCount === 0) return;

                const range = selection.getRangeAt(0); // Текущий диапазон выделения
                const currentCursorPosition = range.startOffset; // Позиция курсора
                // Важно: Используем $nextTick, чтобы дождаться обновления DOM
                this.$nextTick(() => {
                    const contentEditableElement = event.target;

                    // Создаем новый диапазон для восстановления позиции курсора
                    const newRange = document.createRange();
                    const textNode = contentEditableElement.childNodes[0]; // Первый текстовый узел

                    if (textNode) {
                        // Устанавливаем позицию курсора
                        const newPosition = this.cursor_pos ? this.cursor_pos + 1 : Math.min(currentCursorPosition, textNode.length);
                        this.cursor_pos = null;
                        newRange.setStart(textNode, newPosition);
                        newRange.collapse(true);

                        // Применяем новый диапазон к выделению
                        selection.removeAllRanges();
                        selection.addRange(newRange);
                    }
                });
            },
            handleEnterDown(event) {
                if (event.key === 'Enter') {
                    const block = this.textBlocks.find((b) => b.id === this.selectedLay);
                    if (!block) return;

                    const bounds = this.getCropperBounds();
                    console.log(block.top + block.height, bounds.height + bounds.top);
                    if (block.top + block.height + block.fontSize * 2 >= bounds.height + bounds.top) {
                        block.top = bounds.top + bounds.height - block.height - block.fontSize * 2;
                        console.log('enter border');
                    }
                    const selection = window.getSelection();
                    const range = selection.getRangeAt(0); // Текущий диапазон выделения
                    const currentCursorPosition = range.startOffset; // Позиция курсора

                    this.cursor_pos = currentCursorPosition;
                }
            },
            deselectBlock() {
                this.selectedBlockId = null; // Снимаем выделение при потере фокуса
            },
            selectBlock(id, event) {
                this.selectedBlockId = id; // Выбираем блок
                this.selectedLay = id;
                this.enableEditing(id);
                this.startDragText(event, id); // Начинаем перетаскивание
            },
            addRectangle() {
                const bounds = this.getCropperBounds();
                const newRectangle = {
                    id: Date.now(),
                    top: bounds.top + bounds.height / 2 - 25,
                    left: bounds.left + bounds.width / 2 - 50,
                    width: 100,
                    height: 50,
                    color: '#000000',
                    opacity: 0.5,
                    zIndex: 1 + this.layers.length,
                    display: 'block'
                };
                this.rectangles.push(newRectangle);
                this.layers.unshift({ name: `Прямоугольник ${this.rectangles.length}`, id: newRectangle.id, type: "rectangle" });
                this.selectedRectangleId = newRectangle.id;
                this.selectedLay = newRectangle.id;
                this.captureState();
            },
            deleteRectangle(id) {
                console.log(id);
                this.rectangles = this.rectangles.filter((r) => r.id !== id);
                this.layers = this.layers.filter((r) => r.id !== id);
                if (this.selectedRectangleId === id) {
                    this.selectedRectangleId = null;
                    this.captureState();
                }
            },
            selectRectangle(id, event) {
                // Проверяем, что событие не связано с изменением размера
                if (!this.isResizing) {
                    this.selectedRectangleId = id;
                    this.selectedLay = id;
                    this.selectedRectangle = this.rectangles.filter(rect => rect.id == id)[0];
                    this.startDragRectangle(event, id);
                }
            },
            startDragRectangle(event, id) {
                const rectangle = this.rectangles.find((r) => r.id === id);
                if (!rectangle) return;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                this.isDraggingRectangle = true;
                this.dragStartX = clientX - rectangle.left;
                this.dragStartY = clientY - rectangle.top;

                this._touchMoveRectangleHandler = this.dragRectangle.bind(this);

                document.addEventListener('mousemove', this.dragRectangle);
                document.addEventListener('touchmove', this._touchMoveRectangleHandler, { passive: false });
                document.addEventListener('mouseup', this.stopDragRectangle);
                document.addEventListener('touchend', this.stopDragRectangle, false);
                event.preventDefault();

            },
            dragRectangle(event) {
                event.preventDefault();
                if (!this.isDraggingRectangle || !this.selectedRectangleId) return;
                if (!(event.touches && event.target.classList.contains('rectangle')) && (!this.isDraggingRectangle || !this.selectedRectangleId)) return;
                const rectangle = this.rectangles.find((r) => r.id === this.selectedRectangleId);
                if (!rectangle) return;

                const bounds = this.getCropperBounds();
                if (!bounds) return;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                let newLeft = clientX - this.dragStartX;
                let newTop = clientY - this.dragStartY;

                // Ограничение по горизонтали
                if (newLeft < bounds.left) {
                    newLeft = bounds.left;
                } 
                if (newLeft + rectangle.width > bounds.width + bounds.left) newLeft = bounds.width + bounds.left - rectangle.width;
                
                // Ограничение по вертикали
                if (newTop < bounds.top) newTop = bounds.top;
                if (newTop + rectangle.height > bounds.height) newTop = bounds.height - rectangle.height;

                if (newLeft < 0) newLeft = 0;
                if (newTop < 0) newTop = 0;

                rectangle.left = newLeft;
                rectangle.top = newTop;
            }, 
            stopDragRectangle() {
                this.isDraggingRectangle = false;
                this.captureState();
                document.removeEventListener('mousemove', this.dragRectangle);
                if (this._touchMoveRectangleHandler) {
                    document.removeEventListener('touchmove', this._touchMoveRectangleHandler);
                    this._touchMoveRectangleHandler = null;
                }
                document.removeEventListener('mouseup', this.stopDragRectangle);
                document.removeEventListener('touchend', this.stopDragRectangle);
            },
            startResizeRectangle(event, position, id) {
                this.isResizing = true;
                this.resizePosition = position;

                const rectangle = this.rectangles.find((r) => r.id === id);
                if (!rectangle) return;

                // Сохраняем начальные значения
                this.initialCropLeft = rectangle.left;
                this.initialCropTop = rectangle.top;
                this.initialCropWidth = rectangle.width;
                this.initialCropHeight = rectangle.height;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                this.startX = clientX;
                this.startY = clientY;
                

                document.addEventListener('mousemove', this.resizeRectangle);
                document.addEventListener('touchmove', (event) => {
                    this.resizeRectangle(event);
                }, { passive: false });
                document.addEventListener('mouseup', this.stopResizeRectangle);
                document.addEventListener('touchend', this.stopResizeRectangle, false);
                event.preventDefault();
            },
            resizeRectangle(event) {
                if (!(event.touches && event.target.classList.contains('handles')) && (!this.isResizing || !this.selectedRectangleId)) return;

                const rectangle = this.rectangles.find((r) => r.id === this.selectedRectangleId);
                if (!rectangle) return;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                const bounds = this.getCropperBounds();
                if (!bounds) return;

                const deltaX = clientX - this.startX;
                const deltaY = clientY - this.startY;

                switch (this.resizePosition) {
                    case 'top-left':
                        if (this.initialCropLeft + deltaX >= bounds.left && this.initialCropTop + (this.initialCropHeight - rectangle.height) >= bounds.top) {
                            rectangle.left = this.initialCropLeft + deltaX;
                            rectangle.top = this.initialCropTop + deltaY;
                            rectangle.width = this.initialCropWidth - deltaX;
                            rectangle.height = this.initialCropHeight - deltaY;
                        }
                        break;

                    case 'top-right':
                        if (this.initialCropTop + (this.initialCropHeight - rectangle.height) >= bounds.top && this.initialCropLeft + this.initialCropWidth + deltaX <= bounds.left + bounds.width) {
                            rectangle.top = this.initialCropTop + deltaY;
                            rectangle.width = this.initialCropWidth + deltaX;
                            rectangle.height = this.initialCropHeight - deltaY;
                        }
                        break;

                    case 'bottom-left':
                        if (this.initialCropLeft - (rectangle.width - this.initialCropWidth) >= bounds.left && this.initialCropTop + this.initialCropHeight + deltaY <= bounds.height) {
                            rectangle.left = this.initialCropLeft + deltaX;
                            rectangle.width = this.initialCropWidth - deltaX;
                            rectangle.height = this.initialCropHeight + deltaY;
                        }
                        break;

                    case 'bottom-right':
                        if (this.initialCropTop + this.initialCropHeight + deltaY <= bounds.height && this.initialCropLeft + this.initialCropWidth + deltaX <= bounds.left + bounds.width) {
                            rectangle.width = this.initialCropWidth + deltaX;
                            rectangle.height = this.initialCropHeight + deltaY;
                        }
                        break;
                }

                // Ограничение минимальных размеров
                if (rectangle.width < 10) {
                    if (this.resizePosition.includes('left')) {
                    rectangle.left -= 10 - rectangle.width; // Корректируем позицию
                    }
                    rectangle.width = 10;
                }

                if (rectangle.height < 10) {
                    if (this.resizePosition.includes('top')) {
                    rectangle.top -= 10 - rectangle.height; // Корректируем позицию
                    }
                    rectangle.height = 10;
                }

                // Ограничение перемещения за пределы контейнера
                if (rectangle.left < 0) rectangle.left = 0;
                if (rectangle.top < 0) rectangle.top = 0;
            },
            stopResizeRectangle() {
                this.isResizing = false;
                this.captureState();
                document.removeEventListener('mousemove', this.resizeRectangle);
                document.removeEventListener('touchmove', this.resizeRectangle);
                document.removeEventListener('mouseup', this.stopResizeRectangle);
                document.removeEventListener('touchend', this.stopResizeRectangle);
            },
            splitTextIntoLines(text) {
                return text.split('\n');
            },
            wrapText(ctx, text, x, y, maxWidth) {
                const words = text.split(' ');
                let line = '';
                let lines = [];

                for (let n = 0; n < words.length; n++) {
                    const testLine = line + words[n] + ' ';
                    const metrics = ctx.measureText(testLine);
                    const testWidth = metrics.width;

                    if (testWidth > maxWidth && n > 0) {
                    lines.push(line.trim());
                    line = words[n] + ' ';
                    } else {
                    line = testLine;
                    }
                }

                lines.push(line.trim());
                return lines;
            },
            getShift() {
                const wrapper = document.querySelector('.vue-rectangle-stencil');
                const computedStyle = window.getComputedStyle(wrapper);
                const leftValue = computedStyle.transform;

                const match = leftValue.match(/matrix\(\s*[-\d.]+,\s*[-\d.]+,\s*[-\d.]+,\s*[-\d.]+,\s*([-\d.]+),\s*([-\d.]+)\s*\)/);

                if (match) {
                    this.shiftX = parseFloat(match[1]);
                    this.shiftY = parseFloat(match[2]);

                    console.log("Смещение по X:", this.shiftX);
                    console.log("Смещение по Y:", this.shiftY);
                } else {
                    console.error("Неверный формат matrix");
                }
            },
            // eslint-disable-next-line no-unused-vars
            defaultSize({ imageSize, visibleArea }) {
                return {
                    width: (visibleArea || imageSize).width,
                    height: (visibleArea || imageSize).height,
                };
            },
            stencilSize({ boundaries }) {
                return {
                    width: boundaries.width - 100,
                    height: boundaries.height - 100,
                }
            },
            // Загрузка изображения
            // loadImage(event) {
            // const file = event.target.files[0];
            // if (file) {
            //     // Получаем расширение файла
            //     const fileName = file.name;
            //     this.fileExtension = fileName.split('.').pop().toLowerCase();

            //     const reader = new FileReader();
            //     reader.onload = (e) => {
            //     this.imageSrc = e.target.result;
            //     };
            //     reader.readAsDataURL(file);
            //     setTimeout(() => {
            //     this.getShift();
            //     const widthImagePage = document.querySelector('.vue-preview__wrapper').offsetWidth; 
            //     const heightImagePage = document.querySelector('.vue-preview__wrapper').offsetHeight;
            //     this.startSizeW = widthImagePage;
            //     this.startSizeH = heightImagePage;
            //     }, 2000);
                
            // }
            // },

            // Обработка изменений в области обрезки
            onCropChange({ coordinates }) {
                this.cord = coordinates;
            },

            // Обрезка изображения

            resize() {
                const result = this.$refs.cropper.getResult().canvas.toDataURL();
                const cords = this.$refs.cropper.getResult().coordinates;
                console.log(cords);
                this.currentImage = result; // тут устанавливаем обрезанное изображение для пользователя
                this.isCroped = false;

                const widthImagePage = document.querySelector('.vue-preview__wrapper').offsetWidth; 
                const heightImagePage = document.querySelector('.vue-preview__wrapper').offsetHeight;
                this.startSizeW = widthImagePage;
                this.startSizeH = heightImagePage;

                this.textBlocks = this.textBlocks.filter(item => item.left >= cords.left && item.top >= cords.top && item.top + item.fontSize <= cords.top + cords.height);
                this.rectangles = this.rectangles.filter(item => item.left >= cords.left && item.top >= cords.top && item.top <= cords.top + cords.height);
                this.images = this.images.filter(item => item.left >= cords.left && item.top >= cords.top && item.top <= cords.top + cords.height);
                this.captureState();
            },

            // crop() {
            //     // eslint-disable-next-line no-unused-vars
            //     const { coordinates, canvas } = this.$refs.cropper.getResult();
            //     if (!canvas) {
            //         console.error('Canvas is not available');
            //         return;
            //     }

            //     // Создаем новый canvas для добавления текстовых блоков
            //     const finalCanvas = document.createElement('canvas');
            //     const ctx = finalCanvas.getContext('2d');

            //     // Устанавливаем размеры нового canvas
            //     finalCanvas.width = canvas.width;
            //     finalCanvas.height = canvas.height;

            //     // Рисуем обрезанное изображение на новом canvas
            //     ctx.drawImage(canvas, 0, 0);

            //     // Получаем исходные размеры изображения
            //     const img = new Image();
            //     img.src = canvas.toDataURL();
            //     this.getShift();

            //     img.onload = () => {
            //         // Вычисляем масштаб изображения
            //         console.log(document.querySelector('.vue-preview__wrapper'));
            //         const widthImagePage = document.querySelector('.vue-preview__wrapper').offsetWidth;
            //         const heightImagePage = document.querySelector('.vue-preview__wrapper').offsetHeight;
            //         this.startSizeW = widthImagePage;
            //         this.startSizeH = heightImagePage;
            //         const scaleFactorX = img.width / widthImagePage;
            //         const scaleFactorY = img.height / heightImagePage;

            //         // Отрисовка прямоугольников
            //         this.rectangles.forEach((rectangle) => {
            //             ctx.fillStyle = rectangle.color;
            //             ctx.globalAlpha = rectangle.opacity; // Прозрачность прямоугольника

            //             const scaledLeft = (rectangle.left - this.shiftX) * scaleFactorX * (widthImagePage / this.startSizeW);
            //             const scaledTop = (rectangle.top - this.shiftY) * scaleFactorY * (heightImagePage / this.startSizeH);
            //             const scaledWidth = rectangle.width * scaleFactorX * (widthImagePage / this.startSizeW);
            //             const scaledHeight = rectangle.height * scaleFactorY * (heightImagePage / this.startSizeH);

            //             ctx.fillRect(scaledLeft, scaledTop, scaledWidth, scaledHeight);
            //         });

            //         // Возвращаем прозрачность к стандартному значению
            //         ctx.globalAlpha = 1;

            //         const imagePromises = this.images.map((image) => {
            //             return new Promise((resolve) => {
            //                 const overlayImg = new Image();
            //                 overlayImg.src = image.src;

            //                 overlayImg.onload = () => {
            //                     const scaledLeft = (image.left - this.shiftX) * scaleFactorX * (widthImagePage / this.startSizeW);
            //                     const scaledTop = (image.top - this.shiftY) * scaleFactorY * (heightImagePage / this.startSizeH);
            //                     const scaledWidth = image.width * scaleFactorX * (widthImagePage / this.startSizeW);
            //                     const scaledHeight = image.height * scaleFactorY * (heightImagePage / this.startSizeH);

            //                     ctx.drawImage(overlayImg, scaledLeft, scaledTop, scaledWidth, scaledHeight);
            //                     resolve(); // Разрешаем Promise после отрисовки изображения
            //                 };

            //                 overlayImg.onerror = () => {
            //                     console.error('Ошибка загрузки изображения:', image.src);
            //                     resolve(); // Разрешаем Promise даже при ошибке
            //                 };
            //             });
            //         });

            //         // Добавляем текстовые блоки
            //         this.textBlocks.forEach((block) => {
            //             ctx.font = `${block.fontStyle} ${block.fontWeight} ${block.fontSize * scaleFactorY * (heightImagePage / this.startSizeH)}px ${block.fontFamily}`;
            //             ctx.fillStyle = block.color;

            //             // Пересчитываем позицию текста с учётом масштаба
            //             let scaledLeft = (block.left - this.shiftX) * scaleFactorX * (widthImagePage / this.startSizeW);
            //             const scaledTop = (block.top - this.shiftY) * scaleFactorY * (heightImagePage / this.startSizeH) + block.fontSize * scaleFactorY * (heightImagePage / this.startSizeH)  * 1.2;

            //             // Разбиваем текст на строки
            //             const lines = this.splitTextIntoLines(block.text);
            //             const lineHeight = block.fontSize * scaleFactorY * (heightImagePage / this.startSizeH); // Высота строки
                        
            //             const textMetrics = ctx.measureText(lines[0]);
            //             const textWidth = textMetrics.width;

            //             if (block.textAlign === 'center') {
            //                 scaledLeft += textWidth / 2; // Центрируем по ширине текста
            //                 ctx.textAlign = 'center';
            //             } else if (block.textAlign === 'right') {
            //                 scaledLeft += textWidth; // Выравниваем по правому краю
            //                 ctx.textAlign = 'right';
            //             } else {
            //                 ctx.textAlign = 'left'; // По умолчанию выравнивание по левому краю
            //             }
            //             // Рисуем каждую строку
            //             lines.forEach((line, index) => {
            //                 ctx.fillText(line, scaledLeft, scaledTop + index * lineHeight);
            //                 console.log(line, scaledLeft, scaledTop + index * lineHeight);
            //             });
            //         });

            //         // Ждём завершения загрузки всех изображений
            //         Promise.all(imagePromises).then(() => {
            //             // Сохраняем результат в Base64
            //             this.croppedImage = finalCanvas.toDataURL(`image/${this.fileExtension}`);
            //             console.log(this.croppedImage);
            //         });
            //     };
            // },
            async crop() {
                // eslint-disable-next-line no-unused-vars
                const { coordinates, canvas } = this.$refs.cropper.getResult();
                if (!canvas) {
                    console.error('Canvas is not available');
                    return Promise.reject('Canvas not available');
                }

                // 💬 Функция авто-переноса текста
                const autoLineBreak = (text, maxWidth, ctx) => {
                    const resultLines = [];

                    // Сначала разбиваем текст по \n — пользовательские переносы
                    const paragraphs = text.split(/\r?\n/);
                    console.log(paragraphs);

                    for (const paragraph of paragraphs) {
                        const words = paragraph.trim().split(' ');
                        console.log(words);
                        let currentLine = '';

                        for (let i = 0; i < words.length; i++) {
                            const word = words[i];
                            const testLine = currentLine + word;
                            const testWidth = ctx.measureText(testLine).width;

                            console.log(word, "|", testLine);

                            if (Math.floor(testWidth) > Math.floor(maxWidth) + 1 && currentLine !== '') {
                                console.log(currentLine, word, testWidth, maxWidth);
                                resultLines.push(currentLine.trim());
                                currentLine = word + ' ';
                            } else {
                                currentLine = testLine + ' ';
                            }
                        }

                        resultLines.push(currentLine.trim());
                    }
                    console.log(resultLines);
                    return resultLines;
                    // return resultLines.filter(item => item != '');
                };



                return new Promise((resolve, reject) => {
                    const finalCanvas = document.createElement('canvas');
                    const ctx = finalCanvas.getContext('2d');
                    finalCanvas.width = canvas.width;
                    finalCanvas.height = canvas.height;
                    ctx.drawImage(canvas, 0, 0);

                    const img = new Image();
                    img.src = canvas.toDataURL();
                    this.getShift();

                    img.onload = async () => {
                        const widthImagePage = document.querySelector('.vue-preview__wrapper').offsetWidth;
                        const heightImagePage = document.querySelector('.vue-preview__wrapper').offsetHeight;
                        this.startSizeW = widthImagePage;
                        this.startSizeH = heightImagePage;
                        const scaleFactorX = img.width / widthImagePage;
                        const scaleFactorY = img.height / heightImagePage;

                        const allElements = [];

                        // Прямоугольники
                        this.rectangles.forEach(rectangle => {
                            allElements.push({
                                type: 'rectangle',
                                zIndex: rectangle.zIndex || 800,
                                draw: () => {
                                    return new Promise(resolve => {
                                        ctx.save();
                                        ctx.fillStyle = rectangle.color;
                                        ctx.globalAlpha = rectangle.opacity;

                                        const scaledLeft = (rectangle.left - this.shiftX) * scaleFactorX;
                                        const scaledTop = (rectangle.top - this.shiftY) * scaleFactorY;
                                        const scaledWidth = rectangle.width * scaleFactorX;
                                        const scaledHeight = rectangle.height * scaleFactorY;

                                        ctx.fillRect(scaledLeft, scaledTop, scaledWidth, scaledHeight);
                                        ctx.restore();
                                        resolve();
                                    });
                                }
                            });
                        });

                        // Изображения
                        for (const image of this.images) {
                            allElements.push({
                                type: 'image',
                                zIndex: image.zIndex || 1,
                                draw: () => {
                                    return new Promise((resolve, reject) => {
                                        const overlayImg = new Image();
                                        overlayImg.crossOrigin = 'anonymous';
                                        overlayImg.src = image.src;

                                        overlayImg.onload = () => {
                                            const scaledLeft = (image.left - this.shiftX) * scaleFactorX;
                                            const scaledTop = (image.top - this.shiftY) * scaleFactorY;
                                            const baseWidth = image.width * scaleFactorX;
                                            const baseHeight = image.height * scaleFactorY;

                                            const scale = image.scale || 1;

                                            ctx.filter = image.filter ? `blur(${image.filter * 2}px)` : 'none';

                                            ctx.save();
                                            ctx.translate(scaledLeft + baseWidth / 2, scaledTop + baseHeight / 2);
                                            ctx.scale(scale, scale);
                                            ctx.drawImage(overlayImg, -baseWidth / 2, -baseHeight / 2, baseWidth, baseHeight);
                                            ctx.restore();
                                            ctx.filter = 'none';

                                            resolve();
                                        };

                                        overlayImg.onerror = () => {
                                            reject(new Error(`Failed to load image: ${image.src}`));
                                        };
                                    });
                                }
                            });
                        }

                        // Текстовые блоки
                        this.textBlocks.forEach(block => {
                            allElements.push({
                                type: 'text',
                                zIndex: block.zIndex || 900,
                                draw: () => {
                                    return new Promise(resolve => {
                                        ctx.font = `${block.fontStyle} ${block.fontWeight} ${block.fontSize * scaleFactorY}px ${block.fontFamily}`;
                                        ctx.fillStyle = block.color;

                                        let scaledLeft = (block.left - this.shiftX) * scaleFactorX;
                                        const scaledTop = (block.top - this.shiftY) * scaleFactorY + block.fontSize * scaleFactorY;
                                        const scaledWidth = block.width * scaleFactorX;

                                        const lineHeight = block.fontSize * scaleFactorY;

                                        console.log('CROP: ', scaledWidth, block.width, scaleFactorX);

                                        const lines = autoLineBreak(block.text, scaledWidth, ctx);

                                        if (block.textAlign === 'center') {
                                            ctx.textAlign = 'center';
                                            scaledLeft += scaledWidth / 2;
                                        } else if (block.textAlign === 'right') {
                                            ctx.textAlign = 'right';
                                            scaledLeft += scaledWidth;
                                        } else {
                                            ctx.textAlign = 'left';
                                        }

                                        lines.forEach((line, index) => {
                                            ctx.fillText(line, scaledLeft, scaledTop + index * lineHeight);
                                        });

                                        resolve();
                                    });
                                }
                            });
                        });

                        allElements.sort((a, b) => a.zIndex - b.zIndex);

                        try {
                            for (const element of allElements) {
                                await element.draw();
                            }
                            this.croppedImage = finalCanvas.toDataURL(`image/${this.fileExtension}`);
                            resolve();
                        } catch (error) {
                            console.error('Ошибка отрисовки:', error);
                            reject(error);
                        }
                    };

                    img.onerror = () => {
                        reject(new Error('Image failed to load'));
                    };
                });
            },



            // Скачивание обрезанного изображения
            async downloadImage() {
                try {
                    await this.crop(); // дождаться полной отрисовки

                    if (!this.croppedImage) {
                        console.error('Обрезанное изображение не готово');
                        return;
                    }

                    const link = document.createElement('a');
                    link.href = this.croppedImage;
                    link.download = `cropped-image.${this.fileExtension}`;
                    link.click();

                    this.croppedImage = null;
                } catch (error) {
                    console.error('Не удалось скачать изображение:', error);
                }
            },


            // Добавление текстового блока
            addTextBlock() {
                const bounds = this.getCropperBounds();
                const newBlock = {
                    id: Date.now(),
                    text: 'Новый текст',
                    top: bounds.top + bounds.height / 2,
                    left: bounds.left + bounds.width / 2,
                    fontSize: 20,
                    color: '#000000',
                    fontWeight: 'normal', // По умолчанию обычный текст
                    fontStyle: 'normal',  // По умолчанию без курсива
                    fontFamily: 'Arial',  // По умолчанию Arial
                    textAlign: 'left',
                    width: 0, 
                    height: 0,
                    zIndex: 1 + this.layers.length,
                    display: 'block'
                };
                this.textBlocks.push(newBlock);
                this.layers.unshift({ name: newBlock.text, id: newBlock.id, type: "text" });
                this.selectedBlockId = newBlock.id; // Выбираем новый блок
                this.selectedLay = newBlock.id;
                this.$nextTick(() => {
                    this.initResizeObserver(newBlock.id); // Инициализируем наблюдатель после добавления блока
                });
                this.captureState();
            },
            // Удаление текстового блока
            deleteBlock(id) {
                console.log(id);
                if (this.resizeObservers[id]) {
                    const blockElement = document.querySelector(`.text-block[data-id="${id}"]`);
                    if (blockElement) {
                        this.resizeObservers[id].unobserve(blockElement);
                    }
                    delete this.resizeObservers[id];
                }   
                this.textBlocks = this.textBlocks.filter((block) => block.id !== id);
                console.log(this.layers);
                const indexToRemove = this.layers.findIndex(item => item.id === id);
                this.layers.splice(indexToRemove, 1);
                console.log(this.layers);
                if (this.selectedBlockId === id) {
                    this.selectedBlockId = null; // Снимаем выделение
                    this.captureState();
                }
            },


            // Начало перетаскивания текстового блока
            startDragText(event, id) {
                const block = this.textBlocks.find((b) => b.id === id);
                if (!block) return;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                this.isDraggingText = true;
                this.dragStartX = clientX - block.left;
                this.dragStartY = clientY - block.top;

                // Сохраняем ссылку на обработчик
                this._touchMoveHandler = this.dragText.bind(this);

                document.addEventListener('mousemove', this.dragText);
                document.addEventListener('touchmove', this._touchMoveHandler, { passive: false });
                document.addEventListener('mouseup', this.stopDragText);
                document.addEventListener('touchend', this.stopDragText, false);
            },
            dragText(event) {
                event.preventDefault();
                if (!(event.touches && event.target.classList.contains('text-block')) && (!this.isDraggingText || !this.selectedBlockId)) return;
  
                const block = this.textBlocks.find((b) => b.id === this.selectedBlockId);
                if (!block) return;

                const bounds = this.getCropperBounds();
                if (!bounds) return;

                const clientX = event.touches ? event.touches[0].pageX : event.pageX;
                const clientY = event.touches ? event.touches[0].pageY : event.pageY;

                // Вычисляем новые координаты
                let newLeft = clientX - this.dragStartX;
                let newTop = clientY - this.dragStartY;

                // if (newLeft < bounds.left) {
                //     newLeft = bounds.left;
                // } 
                // if (newLeft + block.width > bounds.width + bounds.left) newLeft = bounds.width + bounds.left - block.width;
                
                // Ограничение по вертикали
                // if (newTop < bounds.top) newTop = bounds.top;
                // if (newTop + block.height > bounds.height) newTop = bounds.height - block.height;

                // if (newLeft < 0) newLeft = 0;
                // if (newTop < 0) newTop = 0;

                block.left = newLeft;
                block.top = newTop;
            },

            // Остановка перетаскивания текстового блока
            stopDragText() {
                this.isDraggingText = false;
                this.captureState();

                document.removeEventListener('mousemove', this.dragText);
                if (this._touchMoveHandler) {
                    document.removeEventListener('touchmove', this._touchMoveHandler);
                    this._touchMoveHandler = null;
                }
                document.removeEventListener('mouseup', this.stopDragText);
                document.removeEventListener('touchend', this.stopDragText);
            },
        },
    };
</script>

<style scoped>
    @font-face {
        font-family: 'PathwayExtreme';
        src: url('@/assets/fonts/PathwayExtreme.ttf');
    }
    @font-face {
        font-family: 'PiazzollaItalic';
        src: url('@/assets/fonts/Piazzolla-Italic.ttf');
    }
    @font-face {
        font-family: 'Montserrat';
        src: url('@/assets/fonts/Montserrat.ttf');
    }
    @font-face {
        font-family: 'Roboto';
        src: url('@/assets/fonts/Roboto.ttf');
    }
    .wrapper {
        width: 100vw;
        height: 100%;
        display: flex;
        justify-content: center;
        /* align-items: center; */
        background: rgba(0, 0, 0, 0.3);
        position: fixed;
        left: 0;
        top: 0;
        z-index: 997;
    }
    .editor {
        width: 1054px;
        display: flex;
        flex-direction: column;
        row-gap: 30px;
        padding: 50px 77px;
        background: #1B1E3D;
        border-radius: 10px;
        overflow-y: auto;
        position: relative;
        @media (max-width: 900px) {
            padding: 40px 13px;
            align-items: center;
        }
    }
    h1 {
        font-size: 40px;
        font-family: 'OpenSans';
        color: white;
        font-weight: bold;
        width: fit-content;
        align-self: center;
        @media (max-width: 900px) {
            font-size: 24px;
        }
    }
    .header {
        width: 100%;
        height: 60px;
        display: flex;
        justify-content: space-between;
    }
    .header_tools {
        max-width: 353px;
        display: flex;
        align-items: center;
        justify-content: space-between;
        column-gap: 40px;
        padding: 13px 40px;
        background: #2F3251;
        border-radius: 10px;
        @media (max-width: 900px) {
            width: 100%;
            max-width: none;
        }
    }
    .header_tools img {
        width: 39px;
        height: 39px;
        cursor: pointer;
    }
    .btn1 {
        max-width: 181px;
        @media (max-width: 900px) {
            max-width: none;
        }
    }
    .btn2 {
        max-width: 141px;
        /* @media (max-width: 900px) {
            max-width: none;
        } */
    }
    .save, .download {
        width: 160px;
        font-size: 16px;
        @media (max-width: 900px) {
            width: 140px;
            height: 40px;
            font-size: 14px;
        }
    }
    .tools {
        display: flex;
        flex-direction: column;
        row-gap: 30px;
    }
    .layers {
        max-width: 353px;
        display: flex;
        flex-direction: column;
        row-gap: 15px;
        padding: 15px 10px;
        background: #2F3251;
        border-radius: 10px;
        @media (max-width: 900px) {
            max-width: none;
            width: 100%;
        }
    }
    .layers h2 {
        font-size: 16.4px;
        color: white;
        font-family: 'OpenSans';
        font-weight: normal;
    }
    .line {
        width: 100%;
        height: 1px;
        background: white;
    }
    .layer-item {
        display: grid;
        grid-template-columns: 1fr 1fr 9fr 1fr;
        align-items: center;
        column-gap: 10px;
        margin-bottom: 5px;
        color: white;
        font-size: 16.4px;
        font-family: 'OpenSans';
        padding: 5px 10px;
        cursor: pointer;
    }
    .eye {
        width: 17px;
        cursor: pointer;
        justify-self: center;
    }
    .lay_row {
        display: flex;
        column-gap: 10px;
        align-items: center;
    }
    .delete_lay {
        width: 17px;
        height: 17px;
        justify-self: end;
    }
    .preview {
        max-width: 40px;
        max-height: 25px;
        width: auto;
        object-fit: cover;
        justify-self: center;
    }
    .preview_rect {
        width: 20px;
        height: 20px;
        background: #282826;
        border: 1px solid white;
        justify-self: center;
    }
    .text_tools {
        display: flex;
        flex-direction: column;
        row-gap: 20px;
    }
    .text_tools_row {
        width: 100%;
        display: flex;
        align-items: center;
        column-gap: 22px;
        white-space: nowrap;
    }
    .long {
        @media (max-width: 1100px) {
            flex-wrap: wrap;
        }
    }
    .color_rectangle {
        display: flex;
        flex-direction: column;
        row-gap: 10px;
        align-items: center;
    }
    h2 {
        font-size: 16.4px;
        color: white;
        font-family: 'OpenSans';
        font-weight: normal;
        @media (max-width: 900px) {
            font-size: 14px;
        }
    }
    .color_rect {
        width: 54px;
        height: 54px;
        background: #282826; /* ЗАМЕНИТЬ НА ДИНАМИКУ В КОДЕ */
        cursor: pointer;
    }
    .rect_opacity {
        display: flex;
        flex-direction: column;
        align-items: center;
        row-gap: 10px;
    }
    .vertical_line {
        height: 72px;
        width: 1px;
        background: white;
        margin-left: 10px;
        margin-right: 10px;
    }
    .text_text {
        cursor: pointer;
        display: flex;
        flex-direction: column;
        align-items: center;
        color: white;
    }
    .text_text span {
        font-size: 58.33px;
        font-family: 'PathwayExtreme';
    }
    .bold {
        font-weight: normal;
    }
    .italic {
        /* font-style: italic; */
        font-size: 63px !important;
        font-family: 'PiazzollaItalic' !important;
        line-height: 1.25;
    }
    .color_text {
        display: flex;
        flex-direction: column;
        align-items: center;
        row-gap: 10px;
        position: relative;
    }
    .color_circle {
        width: 45px;
        height: 45px;
        border-radius: 50%;
        border: none;
        transition: 0.1s ease;
    }
    .rect_op {
        width: 32px;
        height: 32px;
        border-radius: 50%;
        border: 1px solid white;
    }
    h3 {
        font-size: 11px;
        color: white;
        font-family: 'OpenSans';
        font-weight: normal;
    }
    .text_align {
        display: flex;
        column-gap: 23px;    
        margin-left: 20px;    
    }
    .text_align_item {
        display: flex;
        flex-direction: column;
        row-gap: 10px;
    }
    .text_align_left, .text_align_center, .text_align_right {
        display: flex;
        flex-direction: column;
        row-gap: 9px;
        cursor: pointer;
    }
    .text_align_center { align-items: center; }
    .text_align_right { align-items: end; }
    .text_align_left div:nth-child(1), .text_align_center div:nth-child(1), .text_align_right div:nth-child(1) {
        width: 45.57px;
        background: white;
        height: 1px;
    }
    .text_align_left div:nth-child(2), .text_align_center div:nth-child(2), .text_align_right div:nth-child(2) {
        width: 27.34px;
        height: 1px;
        background: white;
    }
    .text_align_left div:nth-child(3), .text_align_center div:nth-child(3), .text_align_right div:nth-child(3) {
        width: 45.57px;
        height: 1px;
        background: white;
    }
    .text_align_left div:nth-child(4), .text_align_center div:nth-child(4), .text_align_right div:nth-child(4){
        width: 27.34px;
        height: 1px;
        background: white;
    }
    .non_active_align {
        opacity: .5;
    }
    .upload_logo {
        width: 180px;
        font-size: 16px;
    }
    .font_size {
        border: none;
        width: 55px;
        padding: 5px 10px;
        font-size: 20px;
        color: white;
        background: #2F3251;
        appearance: none; /* Убирает стандартные стили браузера */
        -moz-appearance: textfield; /* Для Firefox */
        text-align: center;
        @media (max-width: 900px) {
            font-size: 16px;
            font-weight: normal;
        }
    }
    .font_size::-webkit-inner-spin-button,
    .font_size::-webkit-outer-spin-button {
    -webkit-appearance: none; /* Убирает стрелки для Webkit-браузеров (Chrome, Safari и т.д.) */
    margin: 0; /* Убирает отступы, если они есть */
    }
    .font_dropdown {
        width: 240px;
        height: 40px;
        background: #2F3251;
        font-size: 15px;
        color: white;
        border: none;
        padding: 5px 13px;
        appearance: none; /* Убираем стандартную стрелку */
        -webkit-appearance: none; /* Для Safari */
        -moz-appearance: none; /* Для Firefox */
        background-image: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 10 6"><path fill="white" d="M0 0l5 6 5-6z"/></svg>'); /* Добавляем собственную стрелку */
        background-repeat: no-repeat;
        background-position: right 10px center; /* Позиционируем стрелку */
        background-size: 10px 6px; /* Размер стрелки */
        @media (max-width: 900px) {
            width: 175px;
            height: 30px;
        }
    }
    .cropper-background {
        background: none;
    }
    .image-background {
        position: absolute;
        width: calc(100% + 20px);
        height: calc(100% + 20px);
        left: -10px;
        top: -10px;
        background-size: cover;
        background-position: 50%;
        filter: blur(5px);
    }
    .crop_msg {
        display: flex;
        align-items: center;
        column-gap: 20px;
        align-self: center;
    }
    .crop {
        width: 100px;
        height: 40px;
    }
    .crop_msg span {
        font-size: 16px;
        color: white;
        font-family: 'OpenSans';
    }

    .close {
        position: absolute;
        right: 30px;
        top: 30px;
        width: 21px;
        height: 21px;
        cursor: pointer;
    }
    .vue-advanced-cropper__background, .vue-advanced-cropper__foreground {
        background: none !important;
    }
    .vue-advanced-cropper__image {
        width: 100% !important;
        height: auto;
    }

    /* Стилизация трека (полосы) */
    .custom-range {
        -webkit-appearance: none; /* Убираем стандартный вид */
        width: 100%;
        height: 1px;
        background: rgba(255, 255, 255, 0.502);
        border-radius: 5px;
        outline: none;
    }
    .custom-range-big {
        width: 195px;
        @media (max-width: 900px) {
            width: 122px;
        }
    }
    .custom-range-sm {
        width: 74px;
    }


    /* Стилизация для thumb (кругляшка) */
    .custom-range::-webkit-slider-thumb {
        -webkit-appearance: none;
        appearance: none;
        width: 18.23px;
        height: 18.23px;
        background: white;
        border-radius: 50%;
        cursor: pointer;
    }
    .custom-range-sm::-webkit-slider-thumb {
        -webkit-appearance: none;
        appearance: none;
        width: 8.67px;
        height: 8.67px;
        background: white;
        border-radius: 50%;
        cursor: pointer;
    }
    .custom-range-sm::-moz-range-thumb {
        width: 8.67px;
        height: 8.67px;
        background: white;
        border-radius: 50%;
        cursor: pointer;
        box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        border: none;
    }

    /* Для Firefox */
    .custom-range-big::-moz-range-thumb {
        width: 18.23px;
        height: 18.23px;
        background: white;
        border-radius: 50%;
        cursor: pointer;
        box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        border: none;
    }

    /* Стилизация трека для Firefox */
    .custom-range::-moz-range-track {
        width: 100%;
        height: 1px;
        background: rgba(255, 255, 255, 0.504);
        border-radius: 5px;
    }


    /* ============================================== */
    /* СТИЛИ С СТАРОГО ФАЙЛА !!!!!!!!!!!!!!!!!!!!!!!!!! */
    /* ============================================== */
    .cropper-container {
        position: relative;
        max-width: 511px;
        max-height: 511px;
        overflow: hidden;
        position: relative;
        display: flex;
        @media (max-width: 900px) {
            max-height: 400px;
        }
        /* margin: 20px auto; */
    }

    .text-blocks {
        position: relative;
    }

    .text-block {
        position: absolute;
        font-family: Arial, sans-serif;
        white-space: pre-wrap; /* Сохраняет переносы строк */
        /* word-wrap: break-word;  */
        pointer-events: auto;
        line-height: 1;
        display: inline-block;
        cursor: pointer;
        max-width: 100%;
    }

    .text-block:hover {
        border: 1px dashed blue;
    }

    .layers-panel {
        margin-top: 20px;
    }


    .controls {
        margin-top: 10px;
    }

    .controls {
        margin-top: 10px;
        display: flex;
        align-items: center;
        gap: 10px;
    }

    .controls input[type="checkbox"] {
        margin-right: 5px;
    }

    .controls select {
        padding: 5px;
        font-size: 14px;
    }
    .rectangle {
        position: absolute;
        pointer-events: auto;
    }
    .overlay-image {
        position: absolute;
        pointer-events: all !important;
        /* user-select: none; */
        /* .vue-advanced-cropper__stretcher pointer-events: none */
        /* .vue-advanced-cropper user-select: none */
    }

    .handle {
        position: absolute;
        width: 10px;
        height: 10px;
        background-color: white;
        border: 1px solid black;
        box-sizing: border-box;
        cursor: pointer;
    }

    .rotate_handle {
        position: absolute;
        top: -35px;
        left: calc(50% - 10px);
        width: 25px;
        height: 25px;
        background-color: rgb(255, 255, 255);
        box-sizing: border-box; 
        display: flex;
        justify-content: center;
        align-items: center;   
        border-radius: 50%;    
        transform-origin: center;
    }
    .rotate_handle:hover {
        cursor: grab;
    }
    .rotate_handle:active {
        cursor: grabbing;
    }
    .rotate_handle img {
        width: 15px;
        height: 15px;
        filter: invert(1);
    }

    .handle.top-left {
        top: -5px;
        left: -5px;
        cursor: nw-resize;
    }   

    .handle.top-right {
        top: -5px;
        right: -5px;
        cursor: ne-resize;
    }

    .handle.bottom-left {
        bottom: -5px;
        left: -5px;
        cursor: sw-resize;
    }

    .handle.bottom-right {
        bottom: -5px;
        right: -5px;
        cursor: se-resize;
    }
    .text-block {
        position: absolute;
        pointer-events: auto;
    }
    .layers-list {
        position: relative;
    }

    .drag-handle {
        font-size: 23.11px;
        font-family: 'OpenSans';
        justify-self: center;
    }
    .ghost {
        opacity: 0.5;
        background: #c8ebfb;
    }
    .active_lay_item {
        background: #4D506F;
        border-radius: 5px;
    }
    

    .adding_btns {
        display: flex;
        column-gap: 30px;
        align-self: center;
        align-items: center;
        flex-wrap: wrap;
        @media (max-width: 900px) {
            display: grid;
            grid-template-columns: 1fr 1fr 1fr 1fr;
            column-gap: 15.78px;
        }
    }
    .adding_btns span {
        font-size: 24px;
        color: white;
        font-weight: bold;
        font-family: 'OpenSans';
    }
    .adding_row {
        display: flex;
        flex-direction: column;
        row-gap: 10px;
        align-self: self-start;
    }
    .adding_btns_text {
        font-size: 18px;
        color: white;
        font-family: 'OpenSans';
        align-self: self-start;
    }
    .row {
        display: flex;
        align-items: center;
        align-self: center;
        column-gap: 30px;
        margin-top: 23px;
        @media (max-width: 900px) {
            column-gap: 17.92px;
            align-self: self-start;
        }
    }
    .row_templates {
        column-gap: 23px !important;
        @media (max-width: 900px) {
            column-gap: 15.78px !important;
            align-self: self-start;
            flex-direction: column !important;
            row-gap: 12px;
            align-items: start;
        }
    }
    .row_templates_sub_row {
        display: grid;
        grid-template-columns: 1fr 1fr 1fr 1fr 1fr;
        column-gap: 15.78px !important;
    }
    .row_btns {
        align-self: center !important;
    }
    .row span {
        font-size: 24px;
        color: white;
        font-family: 'OpenSans';
        font-weight: bold;
        @media (max-width: 900px) {
            font-size: 20.52px;
        }
    }
    .main {
        display: grid;
        grid-template-columns: 1fr 1fr;
        column-gap: 38px;
    }
    .cropper_wrapper {
        display: flex;
        flex-direction: column;
        row-gap: 17px;
    }
    .square_btn {
        width: 60px;
        height: 60px;
        display: flex;
        justify-content: center;
        align-items: center;
        @media (max-width: 900px) {
            width: 15vw !important;
            height: 15vw !important;
            font-size: 18.93 !important;
        }
    }
    .not_active {
        background: none !important;
        border: 1px solid white;
    }
    .top_align {
        align-items: start !important;
    }
    .layer_name {
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }
    .emoji-panel {
        position: absolute;
        background: white;
        border: 1px solid #ccc;
        display: flex;
        flex-wrap: wrap;
        padding: 5px;
        width: 180px;
        max-height: 150px;
        overflow-y: auto;
        z-index: 1000;
        left: 60%;
    }

    .emoji {
        font-size: 20px;
        margin: 4px;
        cursor: pointer;
    }

    .emoji_btn {
        width: 50px;
        height: 50px;
        font-size: 30px;
    }
    .modal_wrapper {
        width: 100vw;
        height: 100%;
        display: flex;
        justify-content: center;
        align-items: center;
        /* align-items: center; */
        background: rgba(0, 0, 0, 0.3);
        position: fixed;
        left: 0;
        top: 0;
        z-index: 999;
    }
    .modal {
        width: 545px;
        display: flex;
        flex-direction: column;
        align-items: center;
        row-gap: 30px;
        padding: 58px;
        background: #1B1E3D;
        border-radius: 10px;
        overflow-y: auto;
        position: relative;
        height: fit-content;
        @media (max-width: 900px) {
            padding: 40px 13px;
            align-items: center;
        }
    }
    .close {
        position: absolute;
        right: 30px;
        top: 30px;
        width: 21px;
        height: 21px;
        cursor: pointer;
    }
    .modal span {
        color: white;
        font-size: 20px;
        font-family: 'OpenSans';
    }
    .modal_row {
        display: flex;
        column-gap: 15px;
    }
    .cancel_btn {
        width: 110px;
        height: 60px;
        @media (max-width: 900px) {
            height: 47px;
        }
    }
    .square_btn2 {
        width: 20.5vw;
        height: 20vw;
        display: flex;
        justify-content: center;
        align-items: center;
        @media (max-width: 410px) {
            width: 20vw;
        }
    }
</style>