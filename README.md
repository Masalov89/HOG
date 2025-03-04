const GAME_CONFIG = { 
    width: window.innerWidth,  // Подгоняем ширину под экран
    height: window.innerHeight, // Подгоняем высоту под экран
    idleLimit: 30,              // Лимит времени бездействия (в секундах) перед показом подсказки
    safeCode: ["4", "2", "7", "9"], // Код для открытия сейфа
    colors: {                    // Цветовые константы для различных состояний
        success: 0x00ff00,      // Зеленый цвет для успешных действий
        error: 0xff0000,        // Красный цвет для ошибок
        hint: 0xffd700,         // Золотой цвет для подсказок
        background: 0x000000    // Черный цвет для фона
    }
};

const locationMap = { // Определение карты локаций с настройками переходов
    'front-house': { 
        name: 'Фасад дома', 
        image: 'front-house', 
        neighbors: ['hall'],
        transitions: {
            'hall': { x: 350, y: 240, width: 60, height: 140 } // Правильные координаты для входной двери
        }
    },
    'hall': { 
        name: 'Холл', 
        image: 'hall', 
        neighbors: ['front-house', 'room'],
        transitions: {
            'front-house': { x: 285, y: 450, width: 225, height: 60 }, // Выход на улицу
            'room': { x: 475, y: 272, width: 70, height: 335 } // Дверь в комнату
        }
    },
    'room': { 
        name: 'Комната', 
        image: 'room', 
        neighbors: ['hall'],
        transitions: {
            'hall': { x: 670, y: 430, width: 60, height: 140 } // Дверь в холл
        }
    }
};

const locationPaths = {
    'front-house': {
        backTo: [], // начальная точка, возврата нет
        forwardTo: ['hall'] // можно идти только в дом
    },
    'hall': {
        backTo: ['front-house'], // возврат на улицу
        forwardTo: ['room'] // можно идти в комнату
    },
    'room': {
        backTo: ['hall'], // возврат в холл
        forwardTo: [] // дальше пути нет
    }
};

class FreeRoamScene extends Phaser.Scene { // Сцена входа в дом - первая сцена игры
    constructor() {
        super({ key: 'FreeRoamScene' }); // Инициализация сцены с уникальным ключом
    }

    preload() { // Предварительная загрузка всех необходимых изображений
        this.load.image('front-house', 'front-house.jpg'); // Загрузка изображения фасада дома
        this.load.image('hall', 'hall.jpg');               // Загрузка изображения холла
        this.load.image('room', 'room.jpg');               // Загрузка изображения комнаты
    }

    create() { // Создание элементов сцены
        this.cameras.main.fadeIn(1000); // Плавное появление сцены за 1 секунду
        this.add.image(GAME_CONFIG.width / 2, GAME_CONFIG.height / 2, 'front-house'); // Добавление фонового изображения фасада дома
        this.createNavigationZones(); // Используем общий механизм создания зон навигации

        // Запускаем сцену инвентаря
        if (!this.scene.isActive('InventoryScene')) {
            this.scene.launch('InventoryScene');
            this.scene.bringToTop('InventoryScene'); // Добавляем эту строку
        }
    }

    createNavigationZones() {
        const neighbors = locationMap['front-house'].neighbors;
        const transitions = locationMap['front-house'].transitions;
    
        neighbors.forEach(neighbor => {
            const zoneName = locationMap[neighbor].name;
            const transition = transitions[neighbor];
    
            const navigationZone = this.add.zone(
                transition.x, 
                transition.y, 
                transition.width, 
                transition.height
            )
            .setOrigin(0.5)
            .setInteractive()
            .on('pointerdown', () => {
                this.cameras.main.fadeOut(1000);
                this.time.delayedCall(1000, () => {
                    this.scene.start('MainScene', { 
                        location: neighbor, 
                        history: ['front-house'] 
                    });
                });
            });
        });
    }
}

class InventorySystem {
    constructor(scene) {
        this.scene = scene;
        this.slots = [];
        this.items = [];
        
        // Настройки инвентаря
        this.config = {
            slots: 5,
            slotSize: 50,
            padding: 10,
            startX: 10, // Зазор в 10 пикселей от левого края
            // Центрирование по высоте экрана
            startY: (GAME_CONFIG.height - ((50 + 10) * 5 - 10)) / 2,
            backgroundColor: 0x333333,
            slotColor: 0x666666,
            borderColor: 0x888888
        };

        this.createInventoryUI(); // Создание UI инвентаря
    }

    createInventoryUI() {
        const bgWidth = this.config.slotSize + this.config.padding * 2;
        const bgHeight = (this.config.slotSize + this.config.padding) * this.config.slots + this.config.padding;

        // Создаем контейнер для инвентаря
        const background = this.scene.add.graphics();
        
        // Фон инвентаря
        background.fillStyle(this.config.backgroundColor, 0.8); // Цвет фона с прозрачностью
        background.fillRoundedRect(
            this.config.startX - this.config.padding,
            this.config.startY - this.config.padding,
            bgWidth,
            bgHeight,
            8
        );

        // Создаем слоты
        for (let i = 0; i < this.config.slots; i++) {
            const x = this.config.startX;
            const y = this.config.startY + i * (this.config.slotSize + this.config.padding);

            // Создаем слот
            const slot = this.scene.add.graphics();
            
            // Фон слота
            slot.fillStyle(this.config.slotColor, 1);
            slot.fillRoundedRect(0, 0, this.config.slotSize, this.config.slotSize, 5);
            
            // Рамка слота
            slot.lineStyle(1, 0xffffff, 0.3);
            slot.strokeRoundedRect(0, 0, this.config.slotSize, this.config.slotSize, 5);
            
            // Добавляем блик
            slot.lineStyle(1, 0xffffff, 0.3);
            slot.lineBetween(2, 2, this.config.slotSize - 2, 2);

            slot.x = x;
            slot.y = y;

            this.slots.push({
                graphics: slot,
                item: null,
                index: i
            });
        }
    }

    addItem(key, name) {
        const emptySlot = this.slots.find(slot => !slot.item);
        if (!emptySlot) return false;
    
        // Создание спрайта предмета
        const item = this.scene.add.sprite(
            emptySlot.graphics.x + this.config.slotSize / 2,
            emptySlot.graphics.y + this.config.slotSize / 2,
            key
        );
        
        const scale = (this.config.slotSize * 0.8) / Math.max(item.width, item.height);
        item.setScale(scale);
    
        item.setInteractive({ draggable: true });
        this.scene.input.setDraggable(item);
        
        // Добавляем обработчик клика для просмотра предмета
        item.on('pointerdown', (pointer, localX, localY, event) => {
            // Предотвращаем начало перетаскивания
            event.stopPropagation();
            
            // Показываем предмет в полном размере
            this.showItemPopup(key, name);
        });
    
        item.on('dragstart', () => {
            item.setTint(0x999999);
            item.setDepth(1000);
        });
    
        item.on('drag', (pointer, dragX, dragY) => {
            item.x = dragX;
            item.y = dragY;
        });
    
        item.on('dragend', (pointer) => {
            item.clearTint();
            item.setDepth(0);
    
            const targetBounds = this.scene.character?.getBounds();
            if (targetBounds && targetBounds.contains(pointer.x, pointer.y)) {
                this.giveItemToCharacter(item, emptySlot);
            } else {
                item.x = emptySlot.graphics.x + this.config.slotSize / 2;
                item.y = emptySlot.graphics.y + this.config.slotSize / 2;
            }
        });
    
        item.on('pointerover', () => {
            const tooltip = this.scene.add.text(
                item.x,
                item.y - 40,
                name,
                {
                    fontSize: '14px',
                    backgroundColor: '#000000',
                    padding: { x: 5, y: 3 },
                    fill: '#ffffff'
                }
            ).setDepth(1000).setOrigin(0.5);
            item.tooltip = tooltip;
        });
    
        item.on('pointerout', () => {
            if (item.tooltip) {
                item.tooltip.destroy();
                item.tooltip = null;
            }
        });
    
        emptySlot.item = {
            sprite: item,
            name: name,
            key: key
        };
    
        return true;
    }

    showItemPopup(key, name) {
        // Создаем группу для попапа
        const popupGroup = this.scene.add.group();
    
        // Создаем затемненный фон
        const background = this.scene.add.rectangle(
            GAME_CONFIG.width / 2,
            GAME_CONFIG.height / 2,
            GAME_CONFIG.width,
            GAME_CONFIG.height,
            0x000000,
            0.7
        ).setInteractive();
        popupGroup.add(background);
    
        // Добавляем изображение предмета
        const image = this.scene.add.image(
            GAME_CONFIG.width / 2,
            GAME_CONFIG.height / 2,
            key
        )
        .setInteractive();
        image.setScale(0.8);
        popupGroup.add(image);
    
        // Добавляем кнопку закрытия
        const closeButton = this.scene.add.text(
            GAME_CONFIG.width - 50,
            50,
            'X',
            {
                fontSize: '32px',
                fill: '#ff0000'
            }
        )
        .setInteractive()
        .setDepth(1001);
        popupGroup.add(closeButton);
    
        // Добавляем название предмета
        const itemName = this.scene.add.text(
            GAME_CONFIG.width / 2,
            50,
            name,
            {
                fontSize: '24px',
                fill: '#ffffff'
            }
        ).setOrigin(0.5, 0);
        popupGroup.add(itemName);
    
        // Закрытие попапа по клику на фон или кнопку
        const closePopup = () => {
            popupGroup.destroy(true);
        };
    
        background.on('pointerdown', closePopup);
        image.on('pointerdown', closePopup);
        closeButton.on('pointerdown', closePopup);
    }
    
    giveItemToCharacter(item, slot) {
        this.scene.tweens.add({
            targets: item,
            alpha: 0,
            scale: 0,
            duration: 300,
            ease: 'Power2',
            onComplete: () => {
                item.destroy();
                slot.item = null;
                if (this.scene.onItemGiven) {
                    this.scene.onItemGiven(item.name);
                }
            }
        });
    }

    removeItem(key) {
        const slot = this.slots.find(slot => slot.item?.key === key);
        if (!slot) return false;

        if (slot.item.sprite.tooltip) {
            slot.item.sprite.tooltip.destroy();
        }
        slot.item.sprite.destroy();
        slot.item = null;
        return true;
    }
}

class InventoryScene extends Phaser.Scene {
    constructor() {
        super({ key: 'InventoryScene' });
        this.inventorySystem = null;
        this.items = []; // Добавляем массив для хранения предметов
    }

    preload() {
        // Загружаем изображения, которые могут быть в инвентаре
        const assets = [
            { key: 'list2', path: 'list2.png' },
            { key: 'object8', path: 'ring.png' }
        ];

        assets.forEach(({ key, path }) => {
            if (!this.textures.exists(key)) {
                this.load.image(key, path);
            }
        });
    }

    create() {
        this.inventorySystem = new InventorySystem(this);
        this.cameras.main.setBackgroundColor('rgba(0, 0, 0, 0)');
        this.scene.bringToTop(); // Добавляем эту строку
        
        // Восстанавливаем предметы
        this.items.forEach(item => {
            this.inventorySystem.addItem(item.key, item.name);
        });
    }

    addItem(key, name) {
        if (this.inventorySystem.addItem(key, name)) {
            this.items.push({ key, name });
            return true;
        }
        return false;
    }

    removeItem(key) {
        if (this.inventorySystem.removeItem(key)) {
            this.items = this.items.filter(item => item.key !== key);
            return true;
        }
        return false;
    }
}

class MainScene extends Phaser.Scene { 
    constructor() {
        super({ key: 'MainScene' });

        this.state = { // Инициализация состояния игры
            remainingObjects: [], // Список оставшихся для поиска предметов
            foundObjects: [],     // Список найденных предметов
            hintAvailable: true,  // Флаг доступности подсказки
            idleTime: 0,         // Счетчик времени бездействия
            selectedDigits: ["0", "0", "0", "0"], // Текущий код сейфа
            ringFound: false,     // Флаг нахождения кольца
            inventory: []         // Добавляем поле inventory
        };

        this.popupGroup = null;      // Группа для всплывающих окон
        this.codeInputGroup = null;  // Группа для ввода кода сейфа
        this.ringObject = null;      // Объект кольца
        this.inventory = null;       // Добавляем поле для системы инвентаря
        this.itemTextObjects = []; // Добавляем это свойство
        this.textBackground = null; // И это свойство
    }

    init(data) { // Инициализация сцены с переданными данными
        this.location = data.location || 'front-house'; // Установка текущей локации
        this.locationHistory = data.history || []; // Инициализация истории локаций
        
        // Восстанавливаем состояние
        if (data.gameState) {
            this.state = {
                ...this.state,
                foundObjects: data.gameState.foundObjects || [],
                remainingObjects: data.gameState.remainingObjects || [],
                ringFound: data.gameState.ringFound || false,
                inventory: data.gameState.inventory || []
            };
        }
    }

    preload() { 
        // Проверка и загрузка основных фоновых изображений
        if (!this.textures.exists('front-house')) {
            this.load.image('front-house', 'front-house.jpg');
        }
        if (!this.textures.exists('hall')) {
            this.load.image('hall', 'hall.jpg');
        }
        if (!this.textures.exists('room')) {
            this.load.image('room', 'room.jpg');
        }
    
        // Массив всех игровых ресурсов для загрузки
        const assets = [
            { key: 'background', path: 'room.jpg' },     // Фон комнаты
            { key: 'object1', path: 'plate.png' },       // Тарелка
            { key: 'object2', path: 'candle.png' },      // Свеча
            { key: 'object3', path: 'art.png' },         // Картина
            { key: 'object4', path: 'tea.png' },         // Сервиз
            { key: 'object5', path: 'list1.png' },       // Скомканный лист
            { key: 'object6', path: 'safe.png' },        // Сейф
            { key: 'object7', path: 'sheet.png' },       // Лист
            { key: 'list2', path: 'list2.png' },         // Развернутый лист
            { key: 'close', path: 'close.png' },         // Кнопка закрытия
            { key: 'opensafe', path: 'opensafe.png' },   // Открытый сейф
            { key: 'object8', path: 'ring.png' },        // Кольцо
            { key: 'arrow', path: 'arrow.png' }          // Добавляем стрелку
        ];
    
        // Загрузка всех ресурсов, если они еще не загружены
        assets.forEach(({ key, path }) => {
            if (!this.textures.exists(key)) {
                this.load.image(key, path);
            }
        });
    
        // Обработчик ошибок загрузки
        this.load.on('loaderror', (file) => {
            console.error('Ошибка загрузки ресурса:', file.src);
        });
    
        // Обработчик завершения загрузки
        this.load.on('complete', () => {
            console.log('Загрузка всех ресурсов завершена');
        });
    }

    create() { // Создание игровой сцены
        this.cameras.main.fadeIn(1000); // Добавление эффекта плавного появления
        this.createBackground(); // Создание основных элементов сцены
        if (this.location === 'room') {
            this.createObjects();    // Создание интерактивных объектов
            this.createUI();         // Создание пользовательского интерфейса
        }

        this.createNavigationZones(); // Создаем зоны навигации между локациями
        this.setupIdleTimer();       // Настройка таймера бездействия

        // Запускаем сцену инвентаря, если она еще не запущена
        if (!this.scene.isActive('InventoryScene')) {
            this.scene.launch('InventoryScene');
            this.scene.bringToTop('InventoryScene'); // Добавляем эту строку
        }
    }

    createBackground() { // Создание фона в зависимости от текущей локации
        const backgroundKey = locationMap[this.location].image;
        if (this.textures.exists(backgroundKey)) {
            this.add.image(GAME_CONFIG.width / 2, GAME_CONFIG.height / 2, backgroundKey);
        } else {
            console.error(`Ошибка: Изображение для ${backgroundKey} не загружено`);
        }
    }

    createNavigationZones() {
        // Получаем данные о соседних локациях и переходах
        const neighbors = locationMap[this.location].neighbors;
        const transitions = locationMap[this.location].transitions;
        const currentLocation = this.location;
    
        neighbors.forEach(neighbor => {
            const zoneName = locationMap[neighbor].name;
            const transition = transitions[neighbor];
    
            // Создаем зону перехода
            const navigationZone = this.add.zone(
                transition.x, 
                transition.y, 
                transition.width, 
                transition.height
            )
            .setOrigin(0.5)
            .setInteractive()
            .on('pointerdown', () => {
                this.cameras.main.fadeOut(1000);
                this.time.delayedCall(1000, () => {
                    this.scene.start('MainScene', {
                        location: neighbor,
                        history: [...this.locationHistory, this.location],
                        gameState: {
                            foundObjects: this.state.foundObjects,
                            remainingObjects: this.state.remainingObjects,
                            ringFound: this.state.ringFound,
                            inventory: this.state.inventory
                        }
                    });
                });
            });
    
            // Проверяем, является ли переход возвратом назад
            const isBackwardDirection = locationPaths[currentLocation].backTo.includes(neighbor);
    
            if (isBackwardDirection) {
                // Отступ от края экрана для определения зоны
                const edgeThreshold = 100;
    
                // Определяем, к какому краю экрана ближе всего зона перехода
                const distanceToLeft = transition.x;
                const distanceToRight = GAME_CONFIG.width - transition.x;
                const distanceToBottom = GAME_CONFIG.height - transition.y;
    
                // Создаем спрайт стрелки
                const arrow = this.add.sprite(transition.x, transition.y, 'arrow')
                    .setDepth(1)
                    .setScale(1);
    
                let angle = 0; // По умолчанию вверх
                let textOffsetY = -15; // Смещение текста
    
                // Определяем направление стрелки
                if (distanceToLeft <= edgeThreshold) {
                    angle = -90; // Стрелка влево
                } else if (distanceToRight <= edgeThreshold) {
                    angle = 90;  // Стрелка вправо
                } else if (distanceToBottom <= edgeThreshold) {
                    angle = 180; // Стрелка вниз
                }
    
                // Устанавливаем угол поворота стрелки
                arrow.setAngle(angle);
    
                // Добавляем название локации
                const text = this.add.text(transition.x, transition.y + textOffsetY, zoneName, {
                    fontSize: '12px',
                    fill: '#ffd700',
                    align: 'center'
                }).setOrigin(0.5);
    
                // Добавляем подсветку при наведении
                navigationZone
                    .on('pointerover', () => {
                        arrow.setTint(0xffff00); // Желтая подсветка
                        text.setTint(0xffff00);
                    })
                    .on('pointerout', () => {
                        arrow.clearTint();
                        text.clearTint();
                    });
            }
        });
    }

    createObjects() {
        const objects = [
            { key: 'object1', x: 502, y: 206, name: 'Тарелка' },
            { key: 'object2', x: 175, y: 306, name: 'Череп-свеча' },
            { key: 'object3', x: 354, y: 106, name: 'Картина' },
            { key: 'object4', x: 451, y: 251, name: 'Сервиз' },
            { key: 'object5', x: 485, y: 370, name: 'Скомканный лист' },
            { key: 'object7', x: 117, y: 407, name: 'Лист' },
            { 
                key: 'object8', 
                x: 353, 
                y: 246, 
                name: 'Кольцо',
                hidden: true,
                dependsOn: 'Сейф',
                available: false
            }
        ];
    
        // Инициализируем remainingObjects, включая скрытые предметы
        if (!this.state.remainingObjects || this.state.remainingObjects.length === 0) {
            this.state.remainingObjects = objects;
        }
    
        // Создаем только видимые объекты, которые еще не были найдены
        this.state.remainingObjects.forEach(obj => {
            // Проверяем, не был ли предмет уже найден и не является ли он скрытым
            if (!obj.hidden && !this.state.foundObjects.includes(obj.name)) {
                this.createGameObject(obj);
            }
        });
    
        // Создаем сейф, только если кольцо еще не найдено
        if (!this.state.ringFound) {
            this.createSafe();
        }
    }

    createGameObject(objData) { // Создание отдельного игрового объекта
        const object = this.add.sprite(objData.x, objData.y, objData.key) // Создание интерактивного спрайта с заданными параметрами
            .setInteractive()
            .setName(objData.name)
            .on('pointerdown', () => this.handleObjectClick(object));

        return object;
    }

    createSafe() { // Создание сейфа
        this.createGameObject({
            key: 'object6',
            x: 353,
            y: 246,
            name: 'Сейф'
        });
    }

    createUI() {
        // Отступ от верхнего края
        const padding = 5;
        
        // 1. Создаем фон сначала
        this.textBackground = this.add.rectangle(
            GAME_CONFIG.width / 2,    // центрируем по горизонтали
            padding + 10,             // небольшой отступ сверху
            320,                      // ширина фона
            200,                      // начальная высота фона
            0x000000,                // черный цвет
            0.8                      // прозрачность 80%
        ).setOrigin(0.5, 0);         // привязка к верхнему краю
        this.textBackground.setDepth(-1); // фон под текстом
    
        // 2. Создаем текстовый контейнер
        this.itemList = this.add.text(
            GAME_CONFIG.width / 2,    // центрируем по горизонтали
            padding + 20,             // отступ от верха + 20 пикселей для читаемости
            '',                       // пустой текст изначально
            {
                fontSize: '13px',
                fill: '#ffffff',
                padding: { x: 15, y: 5 },
                align: 'left',        // выравнивание по левому краю
                fixedWidth: 300,      // фиксированная ширина
                fixedHeight: 180,     // фиксированная высота
                backgroundColor: null  // убираем фон у текста
            }
        ).setOrigin(0.5, 0);         // центрирование по горизонтали
    
        // Разделяем предметы на две колонки
        const itemsPerColumn = Math.ceil(this.state.remainingObjects.length / 2);
        const leftColumnItems = this.state.remainingObjects.slice(0, itemsPerColumn);
        const rightColumnItems = this.state.remainingObjects.slice(itemsPerColumn);
    
        // Создаем массивы для обеих колонок с учетом цвета
        const leftColumn = leftColumnItems.map(obj => {
            const color = (obj.hidden && !obj.available) ? '#ff0000' : '#ffffff';
            return { 
                text: obj.name, 
                color: color 
            };
        });
        
        const rightColumn = rightColumnItems.map(obj => {
            const color = (obj.hidden && !obj.available) ? '#ff0000' : '#ffffff';
            return { 
                text: obj.name, 
                color: color 
            };
        });
    
        // Находим максимальную длину имени предмета для выравнивания колонок
        const maxLength = Math.max(
            ...leftColumn.map(item => item.text.length),
            ...rightColumn.map(item => item.text.length)
        );
    
        // Формируем текст для списка
        let formattedText = '';
        leftColumn.forEach((leftItem, index) => {
            const rightItem = rightColumn[index] || { text: '', color: '#ffffff' };
            formattedText += leftItem.text.padEnd(maxLength + 5);
            if (rightItem.text) {
                formattedText += rightItem.text;
            }
            if (index < leftColumn.length - 1) {
                formattedText += '\n';
            }
        });
    
        // Устанавливаем текст в контейнер
        this.itemList.setText(formattedText);
    
        // Обновляем список предметов
        this.updateItemList();
        
        // Создаем кнопку подсказки
        this.createHintButton();
    
        return this.itemList;
    }

    createHintButton() {
        // Уменьшаем отступ и размер кнопки
        const padding = 5; // Уменьшаем отступ до 10 пикселей
        const buttonWidth = 100; // Делаем кнопку чуть уже
        const buttonHeight = 35; // И чуть ниже
        
        // Вычисляем координату X (максимально близко к правому краю)
        const buttonX = GAME_CONFIG.width - (buttonWidth / 2) - padding;
        // Y координата - минимальный отступ сверху
        const buttonY = (buttonHeight / 2) + padding;
    
        // Создаем фон кнопки
        this.hintButton = this.add.rectangle(
            buttonX,
            buttonY,
            buttonWidth,
            buttonHeight,
            GAME_CONFIG.colors.background
        ).setInteractive();
        
        // Добавляем текст, немного уменьшаем шрифт
        this.hintText = this.add.text(
            buttonX,
            buttonY,
            'Подсказка',
            {
                fontSize: '18px', // Уменьшаем размер шрифта
                fill: '#fff'
            }
        ).setOrigin(0.5);
    
        // Обработчик нажатия остается тем же
        this.hintButton.on('pointerdown', () => {
            if (this.state.hintAvailable) {
                this.useHint();
            }
        });
    }

    handleObjectClick(object) {
        if (!object || object.isAnimating) {
            return;
        }
    
        object.isAnimating = true;
    
        // Добавляем обработку зависимых предметов
        this.handleDependentObjects(object.name);
    
        switch (object.name) {
            case 'Сейф':
                object.isAnimating = false;
                this.openSafe();
                break;
            case 'Скомканный лист':
                this.showPopup(object, 'list2', true);
                break;
            case 'Кольцо':
                const inventoryScene = this.scene.get('InventoryScene');
                inventoryScene.addItem('object8', 'Кольцо');
                this.animateObjectDisappearance(object);
                break;
            default:
                this.animateObjectDisappearance(object);
        }
    }
    
    handleDependentObjects(parentObjectName) {
        this.state.remainingObjects.forEach(obj => {
            if (obj.dependsOn === parentObjectName) {
                this.updateItemList(); // Обновляем список предметов
                console.log(`Предмет ${obj.name} стал доступен после взаимодействия с ${parentObjectName}`);
            }
        });
    }

    animateObjectDisappearance(object) {
        if (!object || !object.scene) {
            return;
        }
    
        object.setTint(GAME_CONFIG.colors.success);
    
        // Анимация увеличения
        this.tweens.add({
            targets: object,
            scale: 1.5,
            duration: 350,
            ease: 'Power1',
            onComplete: () => {
                // Анимация исчезновения
                this.tweens.add({
                    targets: object,
                    alpha: 0,
                    duration: 350,
                    ease: 'Power1',
                    onComplete: () => {
                        if (object.name === 'Кольцо') {
                            this.state.ringFound = true;
                        }
                        
                        if (!this.state.foundObjects.includes(object.name)) {
                            this.state.foundObjects.push(object.name);
                        }
                        
                        this.updateItemList();
                        
                        if (object.scene) {
                            object.destroy();
                        }
                    }
                });
            }
        });
    
        this.state.idleTime = 0;
        this.hintButton.clearTint();
        this.hintText.setStyle({ fill: '#fff' });
    }
    
    showPopup(object, imageKey, addToInventory = false) {
        if (this.popupGroup) {
            this.closePopup();
        }
    
        this.popupGroup = this.add.group();
    
        const popupBackground = this.add.rectangle(350, 241, 700, 482, 0x000000, 0.5)
            .setDepth(998);
        this.popupGroup.add(popupBackground);
    
        const popupImage = this.add.image(350, 241, imageKey)
            .setInteractive()
            .setDepth(999);
        popupImage.setScale(0.8);
        this.popupGroup.add(popupImage);
    
        popupImage.on('pointerdown', () => {
            if (addToInventory && imageKey === 'list2') {
                const inventoryScene = this.scene.get('InventoryScene');
                inventoryScene.addItem('list2', 'Развернутый лист');
            }
            this.closePopup();
            if (object) {
                this.animateObjectDisappearance(object);
            }
        });
    
        // Добавляем обработчик для сброса флага при закрытии попапа
        popupBackground.on('pointerdown', () => {
            if (object) {
                object.isAnimating = false;
            }
            this.closePopup();
        });
    }

    reopenPopup(scene) { // Повторное открытие попапа (например, для просмотра предмета из инвентаря)
        if (!scene) return;

        this.popupGroup = scene.add.group();

        let popupBackground = scene.add.rectangle(350, 241, 700, 482, 0x000000, 0.5); // Создание затемненного фона
        this.popupGroup.add(popupBackground);

        let popupImage = scene.add.image(350, 241, 'list2').setInteractive(); // Добавление изображения
        popupImage.setScale(0.8, 0.8);
        this.popupGroup.add(popupImage);

        popupImage.on('pointerdown', () => this.closePopup()); // Обработчик закрытия
    }

    closePopup() { // Закрытие всплывающего окна 
        if (this.popupGroup) {
            this.popupGroup.getChildren().forEach(child => child.removeAllListeners()); // Удаление всех обработчиков событий и очистка группы
            this.popupGroup.clear(true, true);
            this.popupGroup = null;
        }
    }

    addToInventory(itemName) { // Добавление предмета в инвентарь
        if (!this.state.inventory.includes(itemName)) {
            this.state.inventory.push(itemName); // Добавление предмета в массив инвентаря

            const inventoryTextContent = this.state.inventory.map(item => `[${item}]`).join(' '); // Обновление отображения инвентаря
            this.inventoryText.setText(inventoryTextContent);

            this.inventoryText.removeAllListeners('pointerdown'); // Добавление интерактивности для предметов в инвентаре
            this.inventoryText.setInteractive(new Phaser.Geom.Rectangle(0, 0, this.inventoryText.width, this.inventoryText.height), Phaser.Geom.Rectangle.Contains);
            this.inventoryText.on('pointerdown', () => {
                if (itemName === 'Развернутый лист') {
                    this.reopenPopup(this.inventoryText.scene);
                }
            });
        }

        this.state.idleTime = 0; // Сброс таймера бездействия
        this.hintButton.clearTint();
        this.hintText.setStyle({ fill: '#fff' });
    }

    openSafe() { // Открытие интерфейса сейфа
        this.codeInputGroup = this.add.group();

        let background = this.add.rectangle(350, 241, 400, 200, 0x000000, 0.8); // Создание фона для интерфейса сейфа
        this.codeInputGroup.add(background);

        let digitTexts = [];

        for (let i = 0; i < 4; i++) { // Создание полей для ввода цифр кода
            let digitText = this.add.text(250 + i * 50, 220, this.state.selectedDigits[i], { 
                fontSize: '32px', 
                fill: '#fff' 
            }).setInteractive();
            
            digitText.index = i;
            digitText.on('pointerdown', () => {
                this.incrementDigit(digitText);
            });
            
            this.codeInputGroup.add(digitText);
            digitTexts.push(digitText);
        }

        let instructions = this.add.text(250, 180, 'Выставьте код:', { // Добавление инструкции
            fontSize: '24px', 
            fill: '#fff' 
        });
        this.codeInputGroup.add(instructions);

        let confirmButton = this.add.text(350, 280, 'ОК', { // Создание кнопки подтверждения
            fontSize: '32px', 
            fill: '#00ff00' 
        }).setInteractive();
        confirmButton.on('pointerdown', () => this.checkCode(digitTexts));
        this.codeInputGroup.add(confirmButton);

        let closeButton = this.add.text(450, 180, 'X', { // Создание кнопки закрытия
            fontSize: '32px', 
            fill: '#ff0000' 
        }).setInteractive();
        closeButton.on('pointerdown', () => this.closeCodeInput());
        this.codeInputGroup.add(closeButton);
    }

    incrementDigit(digitText) { // Увеличение значения цифры в коде сейфа
        let currentDigit = parseInt(digitText.text);
        if (isNaN(currentDigit)) currentDigit = 0;
        currentDigit = (currentDigit + 1) % 10;  // Циклическое изменение от 0 до 9
        digitText.setText(currentDigit.toString());
        this.state.selectedDigits[digitText.index] = currentDigit.toString();
    }

    checkCode(digitTexts) {
        try {
            // Получаем введенный код и правильный код
            const enteredCode = this.state.selectedDigits.join('');
            const correctCode = GAME_CONFIG.safeCode.join('');
            console.log('Введенный код:', enteredCode);
            console.log('Правильный код:', correctCode);
    
            if (enteredCode === correctCode) {
                // Визуальная обратная связь - подсветка правильного кода зеленым
                digitTexts.forEach(digitText => 
                    digitText.setFill('#' + GAME_CONFIG.colors.success.toString(16))
                );
    
                // Закрываем интерфейс ввода кода
                this.closeCodeInput();
    
                // Создаем группу для открытого сейфа и кольца
                const safeGroup = this.add.group();
    
                // Создаем затемненный фон
                const safeBackground = this.add.rectangle(
                    350, 241,    // координаты центра
                    700, 482,    // размеры
                    0x000000,    // цвет
                    0.5         // прозрачность
                ).setDepth(1000);
                safeGroup.add(safeBackground);
    
                // Показываем изображение открытого сейфа
                const openSafeImage = this.add.image(350, 241, 'opensafe')
                    .setScale(0.8)
                    .setDepth(1001)
                    .setInteractive();
                safeGroup.add(openSafeImage);
    
                // Находим объект кольца в списке оставшихся предметов
                const ringObject = this.state.remainingObjects.find(obj => obj.name === 'Кольцо');
                
                // Если кольцо найдено в списке, делаем его доступным
                if (ringObject) {
                    ringObject.available = true;
                    ringObject.hidden = false; // Убираем флаг скрытости
                    this.updateItemList(); // Обновляем список предметов (цвет изменится на белый)
                }
    
                // Если кольцо еще не было найдено игроком
                if (!this.state.ringFound) {
                    console.log('Создаем кольцо...');
                    // Создаем спрайт кольца в открытом сейфе
                    this.ringObject = this.add.sprite(353, 236, 'object8')
                        .setDepth(1002)
                        .setInteractive();
                    this.ringObject.name = 'Кольцо';
                    
                    // Добавляем обработчик клика на кольцо
                    this.ringObject.on('pointerdown', () => {
                        console.log('Клик по кольцу');
                        this.handleObjectClick(this.ringObject);
                        safeGroup.destroy(true); // Закрываем сейф после взятия кольца
                    });
    
                    safeGroup.add(this.ringObject);
                    console.log('Кольцо создано:', this.ringObject);
                }
    
                // Добавляем обработчик закрытия сейфа
                openSafeImage.on('pointerdown', () => {
                    if (this.state.ringFound) {
                        safeGroup.destroy(true); // Закрываем сейф только если кольцо уже найдено
                    }
                });
    
                // Удаляем записку с кодом из инвентаря через InventoryScene
                const inventoryScene = this.scene.get('InventoryScene');
                inventoryScene.removeItem('list2');
    
            } else {
                // Визуальная обратная связь - подсветка неправильного кода красным
                digitTexts.forEach(digitText => 
                    digitText.setFill('#' + GAME_CONFIG.colors.error.toString(16))
                );
                
                // Через секунду сбрасываем код и возвращаем белый цвет
                setTimeout(() => {
                    digitTexts.forEach(digitText => digitText.setFill('#fff'));
                    this.resetDigits(digitTexts);
                }, 1000);
            }
        } catch (error) {
            // Обработка возможных ошибок
            console.error('Ошибка в методе checkCode:', error);
            console.error(error.stack);
        }
    }

    resetDigits(digitTexts) { // Сброс введенных цифр кода
        this.state.selectedDigits = ["0", "0", "0", "0"];
        digitTexts.forEach((digitText, i) => {
            digitText.setText("0");
            this.state.selectedDigits[i] = "0";
        });
    }

    closeCodeInput() { // Закрытие интерфейса ввода кода
        if (this.codeInputGroup) {
            this.codeInputGroup.clear(true, true);
            this.codeInputGroup = null;
        }
    }

    updateItemList() {
        // Проверяем существование списка предметов
        if (!this.itemList) return;
    
        // Делим предметы на две колонки
        const itemsPerColumn = Math.ceil(this.state.remainingObjects.length / 2);
        const leftColumnItems = this.state.remainingObjects.slice(0, itemsPerColumn);
        const rightColumnItems = this.state.remainingObjects.slice(itemsPerColumn);
    
        // Очищаем существующие текстовые объекты, если они есть
        if (this.itemTextObjects) {
            this.itemTextObjects.forEach(textObj => textObj.destroy());
        }
        this.itemTextObjects = [];

        const topPadding = 5; // Минимальный отступ сверху
        const itemSpacing = 20; // Расстояние между элементами

        // Создаем или обновляем фон
        if (!this.textBackground) {
            this.textBackground = this.add.rectangle(
                GAME_CONFIG.width / 2,
                topPadding,
                320,
                200,
                0x000000,
                0.8
            ).setOrigin(0.5, 0);
        }
        
        this.textBackground.setDepth(0);

        // Создаем отдельные текстовые объекты для каждого предмета
        leftColumnItems.forEach((item, index) => {
            // Позиция для левой колонки
            const x = GAME_CONFIG.width / 2 - 140;
            const y = topPadding + 10 + (index * itemSpacing);
            
            // Определяем стиль текста в зависимости от состояния предмета
            const textStyle = {
                fontSize: '13px',
                fill: this.getItemColor(item),
                textDecoration: this.state.foundObjects.includes(item.name) ? 'line-through' : ''
            };
            
            // Создаем текстовый объект для левого элемента
            const textItem = this.add.text(x, y, item.name, textStyle).setDepth(1);
            
            // Если предмет найден, добавляем зачеркивание
            if (this.state.foundObjects.includes(item.name)) {
                const strikethrough = this.add.graphics();
                strikethrough.lineStyle(1, 0x808080); // Серая линия
                strikethrough.lineBetween(
                    x, 
                    y + textItem.height / 2,
                    x + textItem.width,
                    y + textItem.height / 2
                );
                strikethrough.setDepth(2);
                this.itemTextObjects.push(strikethrough);
            }
            
            this.itemTextObjects.push(textItem);
            
            // Если есть правый элемент, создаем его тоже
            if (rightColumnItems[index]) {
                const rightItem = rightColumnItems[index];
                const rightX = GAME_CONFIG.width / 2 + 20;
                
                // Определяем стиль текста для правого элемента
                const rightTextStyle = {
                    fontSize: '13px',
                    fill: this.getItemColor(rightItem),
                    textDecoration: this.state.foundObjects.includes(rightItem.name) ? 'line-through' : ''
                };
                
                // Создаем текстовый объект для правого элемента
                const rightTextItem = this.add.text(rightX, y, rightItem.name, rightTextStyle).setDepth(1);
                
                // Если правый предмет найден, добавляем зачеркивание
                if (this.state.foundObjects.includes(rightItem.name)) {
                    const strikethrough = this.add.graphics();
                    strikethrough.lineStyle(1, 0x808080); // Серая линия
                    strikethrough.lineBetween(
                        rightX,
                        y + rightTextItem.height / 2,
                        rightX + rightTextItem.width,
                        y + rightTextItem.height / 2
                    );
                    strikethrough.setDepth(2);
                    this.itemTextObjects.push(strikethrough);
                }
                
                this.itemTextObjects.push(rightTextItem);
            }
        });
    
        // Очищаем старый текст в основном контейнере
        this.itemList.setText('');
    
        // Обновляем позицию основного контейнера
        this.itemList.setPosition(GAME_CONFIG.width / 2, topPadding);
        this.itemList.setOrigin(0.5, 0);

        // Рассчитываем высоту содержимого
        const contentHeight = (leftColumnItems.length * itemSpacing) + 20;
        
        // Обновляем размер и позицию фона
        this.textBackground.setSize(320, contentHeight);
        this.textBackground.setPosition(
            GAME_CONFIG.width / 2,
            topPadding
        );
    }

    // Добавьте этот вспомогательный метод в класс
    getItemColor(item) {
    if (this.state.foundObjects.includes(item.name)) {
        return '#808080'; // Серый цвет для найденных предметов
    }
    if (item.hidden && !item.available) {
        return '#ff0000'; // Красный цвет для скрытых недоступных предметов
    }
    return '#ffffff'; // Белый цвет для обычных предметов
    }

    removeFromInventory(itemName) { // Удаление предмета из инвентаря
        this.state.inventory = this.state.inventory.filter(item => item !== itemName);
        const inventoryTextContent = this.state.inventory.map(item => `[${item}]`).join(' ');
        this.inventoryText.setText(inventoryTextContent);
        this.inventoryText.removeAllListeners('pointerdown');
    }

    useHint() { // Использование подсказки
        if (this.state.remainingObjects.length > 0) {
            const randomIndex = Math.floor(Math.random() * this.state.remainingObjects.length); // Выбор случайного объекта для подсказки
            const hintObject = this.state.remainingObjects[randomIndex];
            const sprite = this.children.list.find(child => child.name === hintObject.name);

            if (sprite) {
                const graphics = this.add.graphics(); // Создание визуальной подсказки
                graphics.lineStyle(2, 0xffffff, 1);
                graphics.strokeRect(sprite.x - sprite.width / 2, sprite.y - sprite.height / 2, 
                                 sprite.width, sprite.height);
                graphics.fillStyle(0xffffff, 0.5);
                graphics.fillRect(sprite.x - sprite.width / 2, sprite.y - sprite.height / 2, 
                                sprite.width, sprite.height);

                this.tweens.add({ // Анимация исчезновения подсказки
                    targets: graphics,
                    alpha: 0,
                    duration: 1000,
                    onComplete: () => {
                        graphics.destroy();
                    }
                });

                this.state.hintAvailable = false; // Установка таймера для следующей подсказки
                setTimeout(() => {
                    this.state.hintAvailable = true;
                }, 10000);
                this.state.idleTime = 0;
                this.hintButton.clearTint();
                this.hintText.setStyle({ fill: '#fff' });
            }
        }
    }

    setupIdleTimer() { // Настройка таймера бездействия
        this.time.addEvent({
            delay: 1000,  // Проверка каждую секунду
            callback: this.updateIdleTime,
            callbackScope: this,
            loop: true
        });
    }

    updateIdleTime() { // Обновление времени бездействия
        this.state.idleTime++;
        if (this.state.idleTime >= GAME_CONFIG.idleLimit && this.state.hintAvailable) {
            this.useHint();  // Автоматический показ подсказки при длительном бездействии
        }
    }

    getItemListText() {
        const itemsPerColumn = Math.ceil(this.state.remainingObjects.length / 2);
        const leftColumnItems = this.state.remainingObjects.slice(0, itemsPerColumn);
        const rightColumnItems = this.state.remainingObjects.slice(itemsPerColumn);
    
        const leftColumn = leftColumnItems.map(obj => obj.name);
        const rightColumn = rightColumnItems.map(obj => obj.name);
        
        const maxLength = Math.max(
            ...leftColumn.map(item => item.length),
            ...rightColumn.map(item => item.length)
        );
    
        return leftColumn.map((item, index) => {
            const rightItem = rightColumn[index] || '';
            return `${item.padEnd(maxLength + 5)}${rightItem}`;
        }).join('\n');
    }
}

const config = {
    type: Phaser.AUTO,
    width: GAME_CONFIG.width,
    height: GAME_CONFIG.height,
    scene: [FreeRoamScene, MainScene, InventoryScene], // Добавляем InventoryScene
    physics: {
        default: 'arcade',
        arcade: {
            debug: false
        }
    }
};

// Запуск игры
const game = new Phaser.Game(config);
