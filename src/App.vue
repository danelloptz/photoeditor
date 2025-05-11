<template>
  <div>
    <!-- Загрузка изображения -->
    <button @click="crop">Crop</button>
    <button @click="downloadImage">Скачать</button>
    <input type="file" @change="loadImage" accept="image/*" ref="image"/>

    <!-- Текстовые блоки -->
    <!-- <div v-if="imageSrc" class="text-blocks">
      
    </div> -->

    <div v-if="selectedBlock" class="controls">
      <!-- Многострочное поле ввода -->
      <textarea
        v-model="selectedBlock.text"
        placeholder="Введите текст"
        rows="4"
        style="width: 100%;"
      ></textarea>
      <input type="color" v-model="selectedBlock.color" />
      <input type="number" v-model="selectedBlock.fontSize" min="10" max="100" />
      
      <!-- Выбор жирности -->
      <label>
        <input type="checkbox" v-model="selectedBlock.fontWeight" true-value="bold" false-value="normal" />
        Жирный
      </label>

      <!-- Выбор курсива -->
      <label>
        <input type="checkbox" v-model="selectedBlock.fontStyle" true-value="italic" false-value="normal" />
        Курсив
      </label>

      <!-- Выбор шрифта -->
      <select v-model="selectedBlock.fontFamily">
        <option value="Arial">Arial</option>
        <option value="Times New Roman">Times New Roman</option>
        <option value="Courier New">Courier New</option>
        <option value="Verdana">Verdana</option>
      </select>

      <select v-model="selectedBlock.textAlign">
        <option value="left">По левому краю</option>
        <option value="center">По центру</option>
        <option value="right">По правому краю</option>
      </select>
    </div>

    <!-- Панель слоёв -->
    <div v-if="imageSrc" class="layers-panel">
      <h4>Слои</h4>
      <div v-for="block in textBlocks" :key="block.id" class="layer-item">
        <span>{{ block.text || 'Без имени' }}</span>
        <button @click="deleteBlock(block.id)">Удалить</button>
      </div>
      <button @click="addTextBlock">Добавить текст</button>
    </div>

    <!-- Панель слоёв -->
    <div v-if="imageSrc" class="layers-panel">
      <h4>Слои</h4>
      <div v-for="rectangle in rectangles" :key="rectangle.id" class="layer-item">
        <span>Прямоугольник</span>
        <button @click="deleteRectangle(rectangle.id)">Удалить</button>
      </div>
      <button @click="addRectangle">Добавить прямоугольник</button>
    </div>

    <!-- Панель управления прямоугольником -->
    <div v-if="selectedRectangle" class="controls">
      <input type="color" v-model="selectedRectangle.color" />
      <input type="range" v-model="selectedRectangle.opacity" min="0" max="1" step="0.01" />
      <input type="number" v-model="selectedRectangle.width" min="10" />
      <input type="number" v-model="selectedRectangle.height" min="10" />
    </div>

    <!-- Панель слоёв -->
    <div v-if="imageSrc" class="layers-panel">
      <h4>Слои</h4>
      <div v-for="image in images" :key="image.id" class="layer-item">
        <span>Изображение</span>
        <button @click="deleteImage(image.id)">Удалить</button>
      </div>
      <button @click="addImage">Добавить изображение</button>
    </div>

    <!-- Панель управления изображением -->
    <div v-if="selectedImage" class="controls">
      <input type="number" v-model="selectedImage.width" min="10" />
      <input type="number" v-model="selectedImage.height" min="10" />
    </div>

    <!-- Контейнер для vue-advanced-cropper -->
    <div v-if="imageSrc" class="cropper-container" ref="container">
      <div
        v-for="block in textBlocks"
        :key="block.id"
        class="text-block"
        :style="{
          top: `${block.top}px`,
          left: `${block.left}px`,
          fontSize: `${block.fontSize}px`,
          color: block.color,
          fontWeight: block.fontWeight,
          fontStyle: block.fontStyle,
          fontFamily: block.fontFamily,
          textAlign: block.textAlign, // Выравнивание текста
          cursor: 'move',
          zIndex: block.id === selectedBlockId ? 950 : 900,
          border: block.id === selectedBlockId ? '2px solid blue' : 'none',
        }"
        @mousedown="selectBlock(block.id, $event)"
        @click="enableEditing(block.id)"
      >
        <!-- Редактируемый текст -->
        <div
          :contenteditable="block.id === editingBlockId"
          @input="updateText(block.id, $event)"
          @blur="disableEditing(block.id)"
          :style="{
            whiteSpace: 'pre-wrap', // Сохраняем переносы строк
            wordWrap: 'break-word', // Переносим длинные слова
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
          zIndex: image.id === selectedImageId ? 10 : 1,
          border: image.id === selectedImageId ? '2px solid blue' : 'none',
        }"
        @mousedown="selectImage(image.id, $event)"
      >
        <div
          v-for="handle in rectangleHandles"
          :key="handle.position"
          :class="['handle', handle.position]"
          @mousedown="startResizeImage($event, handle.position, image.id)"
        ></div>
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
          zIndex: rectangle.id === selectedRectangleId ? 850 : 800,
          border: rectangle.id === selectedRectangleId ? '2px solid blue' : 'none',
        }"
        @mousedown="selectRectangle(rectangle.id, $event)"
      >
        <div
          v-if="rectangle.id === selectedRectangleId"
          v-for="handle in rectangleHandles"
          :key="handle.position"
          :class="['handle', handle.position]"
          @mousedown="startResizeRectangle($event, handle.position, rectangle.id)"
        ></div>
      </div>
      <cropper
        ref="cropper"
        :src="imageSrc"
        @change="onCropChange"
        :style="{
          maxWidth: '80vw',
          maxHeight: '100vh'
        }"
        :default-size="defaultSize"
      />
    </div>
  </div>
</template>

<script>
import { Cropper } from 'vue-advanced-cropper';
import 'vue-advanced-cropper/dist/style.css';

export default {
  components: {
    Cropper,
  },
  data() {
    return {
      imageSrc: null, // Исходное изображение
      croppedImage: null, // Обрезанное изображение
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
      selectedRectangle: null,
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
    };
  },
  mounted() {
    document.addEventListener('keydown', this.handleKeyDown);
    document.addEventListener('keyup', this.handleKeyUp);
  },
  computed: {
    selectedBlock() {
      return this.textBlocks.find((block) => block.id === this.selectedBlockId);
    },
  },
  methods: {
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
      if (!this.isResizingImage) {
        this.selectedImageId = id;
        this.startDragImage(event, id);
      }
    },
    startDragImage(event, id) {
      const image = this.images.find((img) => img.id === id);
      if (!image) return;
      this.isDraggingImage = true;
      this.dragStartX = event.clientX - image.left;
      this.dragStartY = event.clientY - image.top;
      document.addEventListener('mousemove', this.dragImage);
      document.addEventListener('mouseup', this.stopDragImage);
    },
    dragImage(event) {
      if (!this.isDraggingImage || !this.selectedImageId) return;
      const image = this.images.find((img) => img.id === this.selectedImageId);
      if (!image) return;
      image.left = event.clientX - this.dragStartX;
      image.top = event.clientY - this.dragStartY;
      if (image.left < 0) image.left = 0;
      if (image.top < 0) image.top = 0;
    },
    stopDragImage() {
      this.isDraggingImage = false;
      document.removeEventListener('mousemove', this.dragImage);
      document.removeEventListener('mouseup', this.stopDragImage);
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
      this.startX = event.clientX;
      this.startY = event.clientY;

      document.addEventListener('mousemove', this.resizeImage);
      document.addEventListener('mouseup', this.stopResizeImage);
    },
    resizeImage(event) {
      if (!this.isResizingImage || !this.selectedImageId) return;

      const image = this.images.find((img) => img.id === this.selectedImageId);
      if (!image) return;

      const deltaX = event.clientX - this.startX;
      const deltaY = event.clientY - this.startY;

      // Сохраняем исходное соотношение сторон
      const aspectRatio = this.initialImageWidth / this.initialImageHeight;

      switch (this.resizePosition) {
        case 'top-left':
          image.width = this.initialImageWidth - deltaX;
          image.height = image.width / aspectRatio; // Пропорциональная высота
          image.left = this.initialImageLeft + deltaX;
          image.top = this.initialImageTop + (this.initialImageHeight - image.height);
          break;

        case 'top-right':
          image.width = this.initialImageWidth + deltaX;
          image.height = image.width / aspectRatio; // Пропорциональная высота
          image.top = this.initialImageTop + (this.initialImageHeight - image.height);
          break;

        case 'bottom-left':
          image.height = this.initialImageHeight + deltaY;
          image.width = image.height * aspectRatio; // Пропорциональная ширина
          image.left = this.initialImageLeft - (image.width - this.initialImageWidth);
          break;

        case 'bottom-right':
          image.width = this.initialImageWidth + deltaX;
          image.height = image.width / aspectRatio; // Пропорциональная высота
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
      document.removeEventListener('mousemove', this.resizeImage);
      document.removeEventListener('mouseup', this.stopResizeImage);
    },
    addImage() {
      const input = document.createElement('input');
      input.type = 'file';
      input.accept = 'image/*';
      input.onchange = (event) => {
        const file = event.target.files[0];
        if (!file) return;

        const reader = new FileReader();
        reader.onload = (e) => {
          const newImage = {
            id: Date.now(),
            top: 50,
            left: 50,
            width: 100,
            height: 100,
            src: e.target.result,
          };
          this.images.push(newImage);
          this.selectedImageId = newImage.id;
        };
        reader.readAsDataURL(file);
      };
      input.click();
    },
    deleteImage(id) {
      this.images = this.images.filter((img) => img.id !== id);
      if (this.selectedImageId === id) {
        this.selectedImageId = null;
      }
    },
    enableEditing(id) {
      this.editingBlockId = id;
    },
    disableEditing() {
      this.editingBlockId = null;
    },
    updateText(id, event) {
      const block = this.textBlocks.find((b) => b.id === id);
      if (block) {
        block.text = event.target.innerText; // Обновляем текст из содержимого элемента
      }
    },
    deselectBlock(id) {
      this.selectedBlockId = null; // Снимаем выделение при потере фокуса
    },
    selectBlock(id, event) {
      this.selectedBlockId = id; // Выбираем блок
      this.startDragText(event, id); // Начинаем перетаскивание
    },
    addRectangle() {
      const newRectangle = {
        id: Date.now(),
        top: 50,
        left: 50,
        width: 100,
        height: 50,
        color: '#000000',
        opacity: 0.5,
      };
      this.rectangles.push(newRectangle);
      this.selectedRectangleId = newRectangle.id;
    },
    deleteRectangle(id) {
      this.rectangles = this.rectangles.filter((r) => r.id !== id);
      if (this.selectedRectangleId === id) {
        this.selectedRectangleId = null;
      }
    },
    selectRectangle(id, event) {
      // Проверяем, что событие не связано с изменением размера
      if (!this.isResizing) {
        this.selectedRectangleId = id;
        this.selectedRectangle = this.rectangles.filter(rect => rect.id == id)[0];
        this.startDragRectangle(event, id);
      }
    },
    startDragRectangle(event, id) {
      const rectangle = this.rectangles.find((r) => r.id === id);
      if (!rectangle) return;
      this.isDraggingRectangle = true;
      this.dragStartX = event.clientX - rectangle.left;
      this.dragStartY = event.clientY - rectangle.top;
      document.addEventListener('mousemove', this.dragRectangle);
      document.addEventListener('mouseup', this.stopDragRectangle);
    },
    dragRectangle(event) {
      if (!this.isDraggingRectangle || !this.selectedRectangleId) return;
      const rectangle = this.rectangles.find((r) => r.id === this.selectedRectangleId);
      if (!rectangle) return;
      rectangle.left = event.clientX - this.dragStartX;
      rectangle.top = event.clientY - this.dragStartY;
      if (rectangle.left < 0) rectangle.left = 0;
      if (rectangle.top < 0) rectangle.top = 0;
    },
    stopDragRectangle() {
      this.isDraggingRectangle = false;
      document.removeEventListener('mousemove', this.dragRectangle);
      document.removeEventListener('mouseup', this.stopDragRectangle);
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

      // Сохраняем начальные координаты мыши
      this.startX = event.clientX;
      this.startY = event.clientY;

      document.addEventListener('mousemove', this.resizeRectangle);
      document.addEventListener('mouseup', this.stopResizeRectangle);
    },
    resizeRectangle(event) {
      if (!this.isResizing || !this.selectedRectangleId) return;

      const rectangle = this.rectangles.find((r) => r.id === this.selectedRectangleId);
      if (!rectangle) return;

      const deltaX = event.clientX - this.startX;
      const deltaY = event.clientY - this.startY;

      switch (this.resizePosition) {
        case 'top-left':
          rectangle.left = this.initialCropLeft + deltaX;
          rectangle.top = this.initialCropTop + deltaY;
          rectangle.width = this.initialCropWidth - deltaX;
          rectangle.height = this.initialCropHeight - deltaY;
          break;

        case 'top-right':
          rectangle.top = this.initialCropTop + deltaY;
          rectangle.width = this.initialCropWidth + deltaX;
          rectangle.height = this.initialCropHeight - deltaY;
          break;

        case 'bottom-left':
          rectangle.left = this.initialCropLeft + deltaX;
          rectangle.width = this.initialCropWidth - deltaX;
          rectangle.height = this.initialCropHeight + deltaY;
          break;

        case 'bottom-right':
          rectangle.width = this.initialCropWidth + deltaX;
          rectangle.height = this.initialCropHeight + deltaY;
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
      document.removeEventListener('mousemove', this.resizeRectangle);
      document.removeEventListener('mouseup', this.stopResizeRectangle);
    },
    splitTextIntoLines(text) {
      return text.split('\n');
    },
    wrapText(ctx, text, x, y, maxWidth, lineHeight) {
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
    defaultSize({ imageSize, visibleArea }) {
			return {
				width: (visibleArea || imageSize).width,
				height: (visibleArea || imageSize).height,
			};
		},
    // Загрузка изображения
    loadImage(event) {
      const file = event.target.files[0];
      if (file) {
        // Получаем расширение файла
        const fileName = file.name;
        this.fileExtension = fileName.split('.').pop().toLowerCase();

        const reader = new FileReader();
        reader.onload = (e) => {
          this.imageSrc = e.target.result;
        };
        reader.readAsDataURL(file);
        setTimeout(() => {
          this.getShift();
          const widthImagePage = document.querySelector('.vue-preview__wrapper').offsetWidth; 
          const heightImagePage = document.querySelector('.vue-preview__wrapper').offsetHeight;
          this.startSizeW = widthImagePage;
          this.startSizeH = heightImagePage;
        }, 2000);
        
      }
    },

    // Обработка изменений в области обрезки
    onCropChange({ coordinates }) {
      this.cord = coordinates;
    },

    // Обрезка изображения
    crop() {
      const { coordinates, canvas } = this.$refs.cropper.getResult();
      if (!canvas) {
        console.error('Canvas is not available');
        return;
      }

      // Создаем новый canvas для добавления текстовых блоков
      const finalCanvas = document.createElement('canvas');
      const ctx = finalCanvas.getContext('2d');

      // Устанавливаем размеры нового canvas
      finalCanvas.width = canvas.width;
      finalCanvas.height = canvas.height;

      // Рисуем обрезанное изображение на новом canvas
      ctx.drawImage(canvas, 0, 0);

      // Получаем исходные размеры изображения
      const img = new Image();
      img.src = this.imageSrc;
      this.getShift();

      img.onload = () => {
        // Вычисляем масштаб изображения
        const widthImagePage = document.querySelector('.vue-preview__wrapper').offsetWidth;
        const heightImagePage = document.querySelector('.vue-preview__wrapper').offsetHeight;
        const scaleFactorX = img.width / widthImagePage;
        const scaleFactorY = img.height / heightImagePage;

        // Отрисовка прямоугольников
        this.rectangles.forEach((rectangle) => {
          ctx.fillStyle = rectangle.color;
          ctx.globalAlpha = rectangle.opacity; // Прозрачность прямоугольника

          const scaledLeft = (rectangle.left - this.shiftX) * scaleFactorX * (widthImagePage / this.startSizeW);
          const scaledTop = (rectangle.top - this.shiftY) * scaleFactorY * (heightImagePage / this.startSizeH);
          const scaledWidth = rectangle.width * scaleFactorX * (widthImagePage / this.startSizeW);
          const scaledHeight = rectangle.height * scaleFactorY * (heightImagePage / this.startSizeH);

          ctx.fillRect(scaledLeft, scaledTop, scaledWidth, scaledHeight);
        });

        // Возвращаем прозрачность к стандартному значению
        ctx.globalAlpha = 1;

        const imagePromises = this.images.map((image) => {
          return new Promise((resolve) => {
            const overlayImg = new Image();
            overlayImg.src = image.src;

            overlayImg.onload = () => {
              const scaledLeft = (image.left - this.shiftX) * scaleFactorX * (widthImagePage / this.startSizeW);
              const scaledTop = (image.top - this.shiftY) * scaleFactorY * (heightImagePage / this.startSizeH);
              const scaledWidth = image.width * scaleFactorX * (widthImagePage / this.startSizeW);
              const scaledHeight = image.height * scaleFactorY * (heightImagePage / this.startSizeH);

              ctx.drawImage(overlayImg, scaledLeft, scaledTop, scaledWidth, scaledHeight);
              resolve(); // Разрешаем Promise после отрисовки изображения
            };

            overlayImg.onerror = () => {
              console.error('Ошибка загрузки изображения:', image.src);
              resolve(); // Разрешаем Promise даже при ошибке
            };
          });
        });

        // Добавляем текстовые блоки
        this.textBlocks.forEach((block) => {
          ctx.font = `${block.fontStyle} ${block.fontWeight} ${block.fontSize * scaleFactorY * (heightImagePage / this.startSizeH)}px ${block.fontFamily}`;
          ctx.fillStyle = block.color;

          // Пересчитываем позицию текста с учётом масштаба
          let scaledLeft = (block.left - this.shiftX) * scaleFactorX * (widthImagePage / this.startSizeW);
          const scaledTop = (block.top - this.shiftY) * scaleFactorY * (heightImagePage / this.startSizeH) + block.fontSize * scaleFactorY * (heightImagePage / this.startSizeH)  * 1.2;

          // Разбиваем текст на строки
          const lines = this.splitTextIntoLines(block.text);
          const lineHeight = block.fontSize * scaleFactorY * (heightImagePage / this.startSizeH); // Высота строки
          
          const textMetrics = ctx.measureText(lines[0]);
          const textWidth = textMetrics.width;

          if (block.textAlign === 'center') {
            scaledLeft += textWidth / 2; // Центрируем по ширине текста
            ctx.textAlign = 'center';
          } else if (block.textAlign === 'right') {
            scaledLeft += textWidth; // Выравниваем по правому краю
            ctx.textAlign = 'right';
          } else {
            ctx.textAlign = 'left'; // По умолчанию выравнивание по левому краю
          }
          // Рисуем каждую строку
          lines.forEach((line, index) => {
            ctx.fillText(line, scaledLeft, scaledTop + index * lineHeight);
          });
        });

        // Ждём завершения загрузки всех изображений
        Promise.all(imagePromises).then(() => {
          // Сохраняем результат в Base64
          this.croppedImage = finalCanvas.toDataURL(`image/${this.fileExtension}`);
        });
      };
    },

    // Скачивание обрезанного изображения
    downloadImage() {
      if (!this.croppedImage) return;

      const link = document.createElement('a');
      link.href = this.croppedImage;
      link.download = `cropped-image.${this.fileExtension}`;
      link.click();
    },

    // Добавление текстового блока
    addTextBlock() {
      const newBlock = {
        id: Date.now(),
        text: 'Новый текст',
        top: 200,
        left: 400,
        fontSize: 20,
        color: '#000000',
        fontWeight: 'normal', // По умолчанию обычный текст
        fontStyle: 'normal',  // По умолчанию без курсива
        fontFamily: 'Arial',  // По умолчанию Arial
        width: 200,           // Ширина текстового блока
        textAlign: 'left',
      };
      this.textBlocks.push(newBlock);
      this.selectedBlockId = newBlock.id; // Выбираем новый блок
    },
    // Удаление текстового блока
    deleteBlock(id) {
      this.textBlocks = this.textBlocks.filter((block) => block.id !== id);
      if (this.selectedBlockId === id) {
        this.selectedBlockId = null; // Снимаем выделение
      }
    },

    // Выбор текстового блока
    selectBlock(id, event) {
      this.selectedBlockId = id;
      this.startDragText(event, id);
    },

    // Начало перетаскивания текстового блока
    startDragText(event, id) {
      const block = this.textBlocks.find((b) => b.id === id);
      if (!block) return;

      this.isDraggingText = true;
      this.dragStartX = event.clientX - block.left;
      this.dragStartY = event.clientY - block.top;

      document.addEventListener('mousemove', this.dragText);
      document.addEventListener('mouseup', this.stopDragText);
    },

    // Перетаскивание текстового блока
    dragText(event) {
      if (!this.isDraggingText || !this.selectedBlockId) return;

      const block = this.textBlocks.find((b) => b.id === this.selectedBlockId);
      if (!block) return;

      block.left = event.clientX - this.dragStartX;
      block.top = event.clientY - this.dragStartY;

      // Ограничение перемещения внутри контейнера
      if (block.left < 0) block.left = 0;
      if (block.top < 0) block.top = 0;
    },

    // Остановка перетаскивания текстового блока
    stopDragText() {
      this.isDraggingText = false;
      document.removeEventListener('mousemove', this.dragText);
      document.removeEventListener('mouseup', this.stopDragText);
    },
  },
};
</script>

<style>
.cropper-container {
  position: relative;
  max-width: 80vw;
  max-height: 100vh;
  /* margin: 20px auto; */
}

.text-blocks {
  position: relative;
}

.text-block {
  position: absolute;
  font-family: Arial, sans-serif;
  white-space: pre-wrap; /* Сохраняет переносы строк */
  word-wrap: break-word; /* Переносит длинные слова */
  pointer-events: auto;
  line-height: normal;
  display: inline-block;
}

.text-block:hover {
  border: 1px dashed blue;
}

.layers-panel {
  margin-top: 20px;
}

.layer-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 5px;
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
  pointer-events: auto;
  user-select: none;
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


</style>