const GAME_CONFIG = { 
    width: 1920,  // Фиксированная ширина Full HD
    height: 1080, // Фиксированная высота Full HD
    idleLimit: 10,              // Лимит времени бездействия (в секундах) перед показом подсказки
    safeCode: ["4", "2", "7", "9"], // Код для открытия сейфа
    colors: {                    // Цветовые константы для различных состояний
        success: 0x00ff00,      // Зеленый цвет для успешных действий
        error: 0xff0000,        // Красный цвет для ошибок
        hint: 0xffd700,         // Золотой цвет для подсказок
        background: 0x000000    // Черный цвет для фона
    }
};

// Инициализация уведомлений
function requestNotificationPermission() {
    if (window.Notification && Notification.permission !== "granted") {
        Notification.requestPermission();
    }
}

// Вызываем функцию при загрузке страницы
if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', requestNotificationPermission);
} else {
    requestNotificationPermission();
}

// Добавляем обработчик клавиши F для переключения полноэкранного режима
document.addEventListener('keydown', function(e) {
    if (e.key === 'f' || e.key === 'F') {
        if (!document.fullscreenElement) {
            // Если сейчас НЕ в полноэкранном режиме - включаем его
            document.documentElement.requestFullscreen();
        } else {
            // Если сейчас В полноэкранном режиме - выключаем его
            document.exitFullscreen();
        }
    }
});

const locationMap = { 
    //'new-location': {
    //    name: 'Новая Локация',
    //    image: 'new-location',
    //    neighbors: ['hall'],
    //    transitions: {
    //        'hall': {
    //            points: [
    //                { x: 300, y: 300 },
    //                { x: 400, y: 300 },
    //                { x: 400, y: 400 },
    //                { x: 300, y: 400 }
    //            ]
    //        }
    //    }
    //}

    'front-house': { 
        name: 'Фасад дома', 
        image: 'front-house', 
        neighbors: ['hall'],
        transitions: {
            'hall': {
                // Массив из 4 точек, определяющих форму зоны
                points: [
                    { x: 925, y: 755 }, // верхняя левая точка
                    { x: 985, y: 755 }, // верхняя правая точка
                    { x: 985, y: 865 }, // нижняя правая точка
                    { x: 925, y: 865 }  // нижняя левая точка
                ]
            }
        }
    },
    'hall': { 
        name: 'Холл', 
        image: 'hall', 
        neighbors: ['front-house', 'leftroom', 'rightroom'],
        transitions: {
            'front-house': {
                points: [
                    { x: 635, y: 1025 },
                    { x: 1355, y: 1025 },
                    { x: 1355, y: 1075 },
                    { x: 635, y: 1075 }
                ]
            },
            'leftroom': {
                points: [
                    { x: 205, y: 360 },
                    { x: 360, y: 415 },
                    { x: 360, y: 970 },
                    { x: 205, y: 1000 }
                ]
            },
            'rightroom': {
                points: [
                    { x: 1680, y: 370 },
                    { x: 1750, y: 290 },
                    { x: 1750, y: 1025 },
                    { x: 1680, y: 980 }
                ]
            },
            //'new-location': {
            //    points: [
            //        { x: 500, y: 500 },
            //        { x: 600, y: 500 },
            //        { x: 600, y: 600 },
            //        { x: 500, y: 600 }
            //    ]
            //}
        }
    },
    'leftroom': { 
        name: 'ЛеваяКомната', 
        image: 'leftroom', 
        neighbors: ['hall'],
        transitions: {
            'hall': {
                points: [
                    { x: 635, y: 1025 },
                    { x: 1355, y: 1025 },
                    { x: 1355, y: 1075 },
                    { x: 635, y: 1075 }
                ]
            }
        }
    },
    'rightroom': { 
        name: 'ПраваяКомната', 
        image: 'rightroom', 
        neighbors: ['hall'],
        transitions: {
            'hall': {
                points: [
                    { x: 635, y: 1025 },
                    { x: 1355, y: 1025 },
                    { x: 1355, y: 1075 },
                    { x: 635, y: 1075 }
                ]
            }
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
        forwardTo: ['leftroom', 'rightroom'] // можно идти в комнату одну из двух
    },
    'leftroom': {
        backTo: ['hall'], // возврат в холл
        forwardTo: [] // дальше пути нет
    },
    'rightroom': {
        backTo: ['hall'],
        forwardTo: []
    },
    //'new-location?': {
    //    backTo: ['hall?'],
    //    forwardTo: [?]
    //},
};

// Утилитный класс для общих методов
class NavigationUtils {
    static createNavigationZones(scene) {
        const currentLocation = scene.location;
        const neighbors = locationMap[currentLocation].neighbors;
        const transitions = locationMap[currentLocation].transitions;

        neighbors.forEach(neighbor => {
            const zoneName = locationMap[neighbor].name;
            const transition = transitions[neighbor];
            const points = transition.points;

            // Создаем зону на основе полигона
            const navigationZone = scene.add.zone(0, 0, 1, 1)
                .setOrigin(0)
                .setInteractive(new Phaser.Geom.Polygon(points), Phaser.Geom.Polygon.Contains);

            // Рисуем видимую границу зоны (для отладки)
            const graphics = scene.add.graphics();
            graphics.lineStyle(2, 0xff0000);
            graphics.beginPath();
            graphics.moveTo(points[0].x, points[0].y);
            
            // Рисуем линии между точками
            points.forEach((point, index) => {
                const nextPoint = points[(index + 1) % points.length];
                graphics.lineTo(nextPoint.x, nextPoint.y);
            });
            
            graphics.closePath();
            graphics.strokePath();

            // Обработка кликов
            navigationZone.on('pointerdown', () => {
                scene.cameras.main.fadeOut(1000);
                scene.time.delayedCall(1000, () => {
                    scene.scene.start('GameScene', {
                        location: neighbor,
                        history: [...(scene.locationHistory || []), currentLocation]
                    });
                });
            });

            // Добавляем стрелки и текст для обратного направления
            const isBackwardDirection = locationPaths[currentLocation].backTo.includes(neighbor);

            if (isBackwardDirection) {
                // Находим центр полигона для размещения стрелки и текста
                const centerX = points.reduce((sum, point) => sum + point.x, 0) / points.length;
                const centerY = points.reduce((sum, point) => sum + point.y, 0) / points.length;

                const edgeThreshold = 100;
                const distanceToLeft = centerX;
                const distanceToRight = GAME_CONFIG.width - centerX;
                const distanceToBottom = GAME_CONFIG.height - centerY;

                const arrow = scene.add.sprite(centerX, centerY, 'arrow')
                    .setDepth(1)
                    .setScale(1);

                let angle = 0;
                let textOffsetY = -15;

                if (distanceToLeft <= edgeThreshold) {
                    angle = -90;
                } else if (distanceToRight <= edgeThreshold) {
                    angle = 90;
                } else if (distanceToBottom <= edgeThreshold) {
                    angle = 180;
                }

                arrow.setAngle(angle);

                const text = scene.add.text(centerX, centerY + textOffsetY, zoneName, {
                    fontSize: '12px',
                    fill: '#ffd700',
                    align: 'center'
                }).setOrigin(0.5);

                navigationZone
                    .on('pointerover', () => {
                        arrow.setTint(0xffff00);
                        text.setTint(0xffff00);
                    })
                    .on('pointerout', () => {
                        arrow.clearTint();
                        text.clearTint();
                    });
            }
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

let globalEnergySystem = null;

class EnergySystem {
    constructor(scene) {
        this.scene = scene;
        
        // Константы системы энергии
        this.INITIAL_ENERGY = 200;
        this.MAX_ENERGY = 200;
        this.REGENERATION_RATE = 1;          
        this.REGENERATION_INTERVAL = 10000;  // 10 секунд
        
        // Стоимость действий
        this.COSTS = {
            SEARCH_ITEM: 5,
            MISS_PENALTY: 10,
            USE_HINT: 10,
            PUZZLE: 20,
            GIVE_ITEM: 5
        };

        // Загружаем текущее состояние
        this.currentEnergy = this.loadEnergy();
        this.lastUpdateTime = this.loadLastUpdateTime();
        
        // Восстанавливаем накопленную энергию
        this.updateEnergy();
        
        // Создаем UI
        this.createEnergyUI();
    }

    updateEnergy() {
        const now = Date.now();
        const timePassed = now - this.lastUpdateTime;
        const energyToAdd = Math.floor(timePassed / this.REGENERATION_INTERVAL);
        
        if (energyToAdd > 0) {
            this.addEnergy(energyToAdd);
            this.lastUpdateTime = now - (timePassed % this.REGENERATION_INTERVAL);
            this.saveLastUpdateTime();
        }
    }

    createEnergyUI() {
        // Оставляем существующий код для позиционирования иконки энергии
        const hintPadding = 5;
        const hintButtonSize = 128;
        const hintButtonX = GAME_CONFIG.width - (hintButtonSize / 2) - hintPadding;
        const energyIconSize = 64;
        const spaceBetweenIcons = 30;
        const energyIconX = hintButtonX - (hintButtonSize / 2) - spaceBetweenIcons - (energyIconSize / 2);
        const energyIconY = (hintButtonSize / 2) + hintPadding;
    
        // Создаем иконку энергии (оставляем как есть)
        this.energyIcon = this.scene.add.sprite(energyIconX, energyIconY, 'energy-icon')
            .setInteractive()
            .setDepth(100)
            .setDisplaySize(energyIconSize, energyIconSize);
    
        // Создаем контейнер для бара в нижней части экрана
        this.energyBar = this.scene.add.graphics()
            .setDepth(100);
    
            this.energyText = this.scene.add.text(
                GAME_CONFIG.width / 2,
                GAME_CONFIG.height - 110 + 15, // Позиция Y будет обновляться в updateEnergyBar
                '', 
                {
                    fontSize: '18px',
                    fill: '#ffffff',
                    fontWeight: 'bold'
                }
            )
            .setDepth(101)  // Поверх бара
            .setOrigin(0.5);  // Центрируем текст
    
        // Создаем кнопку для просмотра рекламы
        this.adButton = this.scene.add.text(
            GAME_CONFIG.width / 2,
            GAME_CONFIG.height - 70,
            'Получить энергию за просмотр рекламы',
            {
                fontSize: '16px',
                fill: '#00ff00',
                backgroundColor: '#333333',
                padding: { x: 15, y: 10 }
            }
        )
        .setInteractive()
        .setDepth(100)
        .setOrigin(0.5)
        .setVisible(false)
        .on('pointerdown', () => this.showRewardedAd())
        .on('pointerover', () => this.adButton.setStyle({ fill: '#ffffff' }))
        .on('pointerout', () => this.adButton.setStyle({ fill: '#00ff00' }));
    
        // Скрываем бар изначально
        this.hideEnergyBar();
    
        // Обработчик клика по иконке (оставляем как есть)
        this.energyIcon.on('pointerdown', () => {
            this.showEnergyBar();
        });
    }
    
    showEnergyBar() {
        // Показываем элементы
        if (this.energyBar) {
            this.energyBar.visible = true;
            this.updateEnergyBar();
        }
        if (this.energyText) {
            this.energyText.visible = true;
        }
        if (this.adButton) {
            this.adButton.visible = true;
        }
    
        // Сначала очищаем существующий таймер, если он есть
        if (this.hideTimer) {
            this.scene.time.removeEvent(this.hideTimer);
        }
    
        // Создаем новый таймер на 5 секунд
        this.hideTimer = this.scene.time.delayedCall(5000, () => {
            this.hideEnergyBar();
        }, [], this);
    }

    hideEnergyBar() {
        // Скрываем элементы
        if (this.energyBar) {
            this.energyBar.visible = false;
        }
        if (this.energyText) {
            this.energyText.visible = false;
        }
        if (this.adButton) {
            this.adButton.visible = false;
        }

        // Очищаем таймер
        if (this.hideTimer) {
            this.scene.time.removeEvent(this.hideTimer);
            this.hideTimer = null;
        }
    }

    updateEnergyBar() {
        this.energyBar.clear();
    
        const barWidth = 300;
        const barHeight = 30;
        const barX = GAME_CONFIG.width / 2 - barWidth / 2;
        const barY = GAME_CONFIG.height - 110;
    
        // Фон бара
        this.energyBar.fillStyle(0x333333, 0.8);
        this.energyBar.fillRoundedRect(barX, barY, barWidth, barHeight, 8);
    
        // Заполнение бара
        const fillWidth = (this.currentEnergy / this.MAX_ENERGY) * barWidth;
        this.energyBar.fillStyle(0x00ff00);
        this.energyBar.fillRoundedRect(barX, barY, Math.min(fillWidth, barWidth), barHeight, 8);
    
        // Рамка бара
        this.energyBar.lineStyle(2, 0xffffff, 0.3);
        this.energyBar.strokeRoundedRect(barX, barY, barWidth, barHeight, 8);
    
        // Обновляем текст и размещаем его по центру бара
        this.energyText.setPosition(
            GAME_CONFIG.width / 2,  // по центру экрана
            barY + (barHeight / 2)  // вертикально по центру бара
        );
        this.energyText.setText(`${this.currentEnergy}/${this.MAX_ENERGY}`);
    }

    hasEnough(amount) {
        this.updateEnergy(); // Обновляем перед проверкой
        return this.currentEnergy >= amount;
    }

    spend(amount, action) {
        this.updateEnergy(); // Обновляем перед тратой
    
        if (this.currentEnergy < amount) {
            this.showNotEnoughEnergy();
            return false;
        }
    
        this.currentEnergy = Math.max(0, this.currentEnergy - amount);
        this.saveEnergy();
        this.updateEnergyBar();
        
        if (this.currentEnergy < 20) {
            this.showLowEnergyWarning();
        }
    
        return true;
    }

    addEnergy(amount) {
        const oldEnergy = this.currentEnergy;
        this.currentEnergy = Math.min(this.MAX_ENERGY, this.currentEnergy + amount);
        
        if (oldEnergy !== this.currentEnergy) {
            this.saveEnergy();
            this.updateEnergyBar();
            
            if (this.currentEnergy >= this.COSTS.SEARCH_ITEM) {
                this.resetEnergyWarning();
            }
            
            if (this.currentEnergy >= this.MAX_ENERGY && oldEnergy < this.MAX_ENERGY) {
                this.sendFullEnergyNotification();
            }
        }
    }

    handleMiss() {
        this.missCount++;
        if (this.missCount >= 5) {
            this.spend(this.COSTS.MISS_PENALTY, 'miss');
            this.missCount = 0;
            this.showFrozenScreen();
        }
    }

    showFrozenScreen() {
        const overlay = this.scene.add.rectangle(
            0, 0, 
            this.scene.game.config.width, 
            this.scene.game.config.height,
            0xffffff, 0.5
        ).setOrigin(0);

        this.scene.time.delayedCall(1000, () => {
            overlay.destroy();
        });
    }

    showNotEnoughEnergy() {
        // Показываем сообщение о недостатке энергии
        const text = this.scene.add.text(
            this.scene.game.config.width / 2,
            this.scene.game.config.height / 2,
            'Недостаточно энергии!',
            {
                fontSize: '24px',
                fill: '#ff0000',
                backgroundColor: '#000000',
                padding: { x: 20, y: 10 }
            }
        ).setOrigin(0.5);

        this.scene.time.delayedCall(2000, () => {
            text.destroy();
        });
    }

    showLowEnergyWarning() {
        // Проверяем, что энергии меньше стоимости минимального действия (SEARCH_ITEM = 5)
        if (this.currentEnergy < this.COSTS.SEARCH_ITEM) {
            // Красим иконку в красный
            this.energyIcon.setTint(0xff0000);
            // Если анимация мигания ещё не создана
            if (!this.blinkTween) {
                this.blinkTween = this.scene.tweens.add({
                    targets: this.energyIcon,
                    alpha: { from: 1, to: 0.5 }, // Мигание от полной видимости до половины
                    duration: 500,
                    yoyo: true, // Анимация будет идти туда-обратно
                    repeat: -1  // Бесконечное повторение
                });
            }
        }
    }

    resetEnergyWarning() {
        // Убираем красный цвет с иконки
        this.energyIcon.clearTint();
        // Если есть анимация мигания - останавливаем её
        if (this.blinkTween) {
            this.blinkTween.stop();
            this.blinkTween.remove();
            this.blinkTween = null;
        }
        // Возвращаем полную видимость иконке
        this.energyIcon.setAlpha(1);
    }

    sendFullEnergyNotification() {
        if ("Notification" in window && Notification.permission === "granted") {
            new Notification("Энергия восстановлена!", {
                body: "Ваша энергия полностью восстановлена!",
                icon: "./UI/ApprovedUI/Energy.png"
            });
        }
    }

    // Методы сохранения/загрузки
    saveEnergy() {
        localStorage.setItem('energy', this.currentEnergy.toString());
    }

    loadEnergy() {
        const saved = localStorage.getItem('energy');
        return saved ? parseInt(saved) : this.INITIAL_ENERGY;
    }

    saveLastUpdateTime() {
        localStorage.setItem('energyLastUpdate', this.lastUpdateTime.toString());
    }

    loadLastUpdateTime() {
        const saved = localStorage.getItem('energyLastUpdate');
        return saved ? parseInt(saved) : Date.now();
    }
    // Добавляем новый метод для показа рекламы
    showRewardedAd() {
    // Здесь будет ваша логика показа рекламы
    const rewardAmount = 50; // количество энергии за просмотр рекламы
    this.addEnergy(rewardAmount);
    
    // Показываем сообщение о получении награды
    const rewardText = this.scene.add.text(
        GAME_CONFIG.width / 2,
        GAME_CONFIG.height / 2,
        `+${rewardAmount} энергии!`,
        {
            fontSize: '32px',
            fill: '#00ff00',
            backgroundColor: '#000000',
            padding: { x: 20, y: 10 }
        }
    )
    .setOrigin(0.5)
    .setDepth(200);

    // Удаляем сообщение через 2 секунды
    this.scene.time.delayedCall(2000, () => {
        rewardText.destroy();
    });
    }
}

class GameScene extends Phaser.Scene { 
    constructor() {
        super({ key: 'GameScene' });

        this.state = { // Инициализация состояния игры
            remainingObjects: [], // Список оставшихся для поиска предметов
            foundObjects: [],     // Список найденных предметов
            hintAvailable: true,  // Флаг доступности подсказки
            idleTime: 0,         // Счетчик времени бездействия
            selectedDigits: ["0", "0", "0", "0"], // Текущий код сейфа
            ringFound: false,     // Флаг нахождения кольца
            inventory: []         // Поле inventory
        };

        // Свойства для навигации
        this.location = 'front-house';
        this.locationHistory = [];

        // Общие свойства
        this.popupGroup = null;      // Группа для всплывающих окон
        this.codeInputGroup = null;  // Группа для ввода кода сейфа
        this.ringObject = null;      // Объект кольца
        this.inventory = null;       // Поле для системы инвентаря
        this.itemTextObjects = [];   // Массив текстовых объектов
        this.textBackground = null;  // Фон для текста

        // Добавляем эти две строки:
        this.lastHintTime = 0;      // Время последней подсказки
        this.hintCooldown = 10000;  // Задержка 10 секунд между подсказками
        // Добавляем свойство для системы энергии
        this.energySystem = null;
    }

    create() {
        this.cameras.main.fadeIn(1000);
        this.createBackground();
    
        // Инициализируем глобальную систему энергии
        if (!globalEnergySystem) {
            globalEnergySystem = new EnergySystem(this);
        } else {
            globalEnergySystem.scene = this;
            globalEnergySystem.createEnergyUI();
        }
        this.energySystem = globalEnergySystem;
    
        // Создаем кнопку подсказки на всех локациях
        this.createHintButton();
    
        // Добавляем обработчик кликов для промахов
        this.input.on('pointerdown', (pointer) => {
            if (!pointer.gameObject) {
                this.energySystem.handleMiss();
            }
        });
    
        // Создаем объекты только в определенной локации
        if (this.location === 'leftroom') {
            this.createObjects();
            this.createUI();
        }
    
        NavigationUtils.createNavigationZones(this);
    
        if (!this.scene.isActive('InventoryScene')) {
            this.scene.launch('InventoryScene');
            this.scene.bringToTop('InventoryScene');
        }
    }

    updateHintButtonState() {
        // Проверяем, существует ли кнопка подсказки
        if (this.hintButton) {
            // Проверяем, достаточно ли энергии для использования подсказки (USE_HINT = 10)
            if (this.energySystem.hasEnough(this.energySystem.COSTS.USE_HINT)) {
                // Если энергии достаточно - кнопка активна
                this.hintButton.clearTint();
                this.hintButton.setInteractive();
            } else {
                // Если энергии недостаточно (9 и меньше) - кнопка неактивна
                this.hintButton.setTint(0x666666); // Серый цвет
                this.hintButton.disableInteractive(); // Отключаем интерактивность
            }
        }
    }

    handleObjectClick(object) {
        if (!object || object.isAnimating) {
            return;
        }

        // Проверяем достаточно ли энергии для действия
        if (!this.energySystem.spend(this.energySystem.COSTS.SEARCH_ITEM, 'search')) {
            return;
        }

        object.isAnimating = true;
        
        // Остальной код handleObjectClick без изменений
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

    useHint() {
        // Проверяем, достаточно ли энергии
        if (!this.energySystem.spend(this.energySystem.COSTS.USE_HINT, 'hint')) {
            return;
        }
    
        const currentTime = Date.now();
        
        // Проверяем, прошло ли 30 секунд с последнего использования
        if (currentTime - this.lastHintTime < this.hintCooldown) {
            // Если не прошло 30 секунд, выходим
            return;
        }
    
        // Обновляем время последней подсказки
        this.lastHintTime = currentTime;
    
        // Показываем подсказку в зависимости от локации
        if (this.location === 'leftroom' && this.state.remainingObjects.length > 0) {
            this.showObjectHint();
        } else {
            this.showNavigationHint();
        }
    }

    init(data) {
        // Объединенная инициализация из обоих классов
        this.location = data.location || 'front-house';
        this.locationHistory = data.history || [];
        
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
        if (!this.textures.exists('leftroom')) {
            this.load.image('leftroom', 'leftroom.jpg');
        }
        if (!this.textures.exists('rightroom')) {
            this.load.image('rightroom', 'rightroom.jpg');
        }
        if (!this.textures.exists('hint-button')) {
        this.load.image('hint-button', './UI/ApprovedUI/Hint.png');
        }
        if (!this.textures.exists('energy-icon')) {
        this.load.image('energy-icon', './UI/ApprovedUI/Energy.png');
        }
        //if (!this.textures.exists('new-location')) {
        //    this.load.image('new-location', 'new-location.jpg');
        //}
        
        // Массив всех игровых ресурсов для загрузки
        const assets = [
            { key: 'background', path: 'leftroom.jpg' },     // Фон комнаты
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
            { key: 'arrow', path: 'arrow.png' }          // Стрелка
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

    createBackground() {
        const backgroundKey = locationMap[this.location].image;
        if (this.textures.exists(backgroundKey)) {
            this.add.image(GAME_CONFIG.width / 2, GAME_CONFIG.height / 2, backgroundKey)
                .setDisplaySize(1920, 1080);
        } else {
            console.error(`Ошибка: Изображение для ${backgroundKey} не загружено`);
        }
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
        const object = this.add.sprite(objData.x, objData.y, objData.key)
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
        
        return this.itemList;
    }

    createHintButton() {
        const padding = 5;
        const buttonSize = 128;
        
        const buttonX = GAME_CONFIG.width - (buttonSize / 2) - padding;
        const buttonY = (buttonSize / 2) + padding;
    
        // Создаем основную кнопку
        this.hintButton = this.add.sprite(
            buttonX,
            buttonY,
            'hint-button'
        )
        .setInteractive()
        .setDisplaySize(buttonSize, buttonSize);
    
        // Функция для создания синхронизированной анимации пульсации
        const createPulseTween = () => {
            const pulseDuration = 700; 
    
            // Объединенная пульсация размера и цвета
            this.hintButtonTween = this.tweens.add({
                targets: this.hintButton,
                scale: { from: 0.92, to: 1.05 }, // Увеличили уменьшение с 0.98 до 0.92
                duration: pulseDuration,
                yoyo: true,
                repeat: -1,
                ease: 'Sine.easeInOut',
                onUpdate: (tween) => {
                    const progress = (tween.getValue() - 0.92) / (1.05 - 0.92); // обновили нормализацию
                    const color = Phaser.Display.Color.Interpolate.ColorWithColor(
                        Phaser.Display.Color.ValueToColor(0xFFFFFF),  // белый
                        Phaser.Display.Color.ValueToColor(0xFFD700),  // золотой
                        100,
                        progress * 100
                    );
                    this.hintButton.setTint(Phaser.Display.Color.GetColor(color.r, color.g, color.b));
                }
            });
        };
    
        // Обработчик клика с анимацией
        this.hintButton.on('pointerdown', () => {
            const currentTime = Date.now();
            if (currentTime - this.lastHintTime >= this.hintCooldown) {
                this.tweens.add({
                    targets: this.hintButton,
                    scale: 0.9,
                    duration: 100,
                    yoyo: true,
                    onComplete: () => {
                        this.useHint();
                        this.hintButton.setAlpha(0.5);
                        this.hintButton.clearTint();
                        if (this.hintButtonTween) {
                            this.hintButtonTween.stop();
                            this.hintButtonTween = null;
                        }
                    }
                });
            } else {
                this.tweens.add({
                    targets: this.hintButton,
                    angle: { from: -5, to: 5 },
                    duration: 100,
                    yoyo: true,
                    repeat: 2,
                    onComplete: () => {
                        this.hintButton.setAngle(0);
                    }
                });
            }
        });
    
        // Проверка доступности подсказки
        this.time.addEvent({
            delay: 1000,
            callback: () => {
                const currentTime = Date.now();
                if (currentTime - this.lastHintTime >= this.hintCooldown) {
                    this.hintButton.setAlpha(1);
                    if (!this.hintButtonTween) {
                        createPulseTween();
                    }
                }
            },
            loop: true
        });
    
        // Начальное состояние
        if (Date.now() - this.lastHintTime < this.hintCooldown) {
            this.hintButton.setAlpha(0.5);
            this.hintButton.clearTint();
        } else {
            createPulseTween();
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
    
    showObjectHint() {
        // Фильтруем список, оставляя только ненайденные предметы
        const availableObjects = this.state.remainingObjects.filter(obj => 
            !this.state.foundObjects.includes(obj.name) && (!obj.hidden || obj.available)
        );
    
        if (availableObjects.length > 0) {
            // Выбираем один случайный предмет из доступных
            const randomIndex = Math.floor(Math.random() * availableObjects.length);
            const hintObject = availableObjects[randomIndex];
            const sprite = this.children.list.find(child => child.name === hintObject.name);
    
            if (sprite) {
                const graphics = this.add.graphics();
                graphics.lineStyle(2, GAME_CONFIG.colors.hint, 1); // Используем золотой цвет
                graphics.strokeRect(sprite.x - sprite.width / 2, sprite.y - sprite.height / 2, 
                                  sprite.width, sprite.height);
                graphics.fillStyle(GAME_CONFIG.colors.hint, 0.3); // Полупрозрачная заливка
                graphics.fillRect(sprite.x - sprite.width / 2, sprite.y - sprite.height / 2, 
                                  sprite.width, sprite.height);
    
                this.tweens.add({
                    targets: graphics,
                    alpha: 0,
                    duration: 1000,
                    onComplete: () => {
                        graphics.destroy();
                    }
                });
            }
        }
    }
    
    showNavigationHint() {
        // Получаем текущую локацию
        const currentLocation = locationPaths[this.location];
        let possibleDirections = [];
        
        // Собираем все возможные направления
        if (currentLocation.forwardTo.length > 0) {
            possibleDirections = possibleDirections.concat(currentLocation.forwardTo);
        }
        if (currentLocation.backTo.length > 0) {
            possibleDirections = possibleDirections.concat(currentLocation.backTo);
        }
    
        if (possibleDirections.length > 0) {
            // Выбираем случайное направление
            const randomDirection = possibleDirections[Math.floor(Math.random() * possibleDirections.length)];
            
            // Находим соответствующую зону перехода
            const transition = locationMap[this.location].transitions[randomDirection];
            
            if (transition) {
                const points = transition.points;
                
                // Создаем подсветку зоны перехода
                const graphics = this.add.graphics();
                graphics.lineStyle(2, GAME_CONFIG.colors.hint, 1);
                graphics.beginPath();
                graphics.moveTo(points[0].x, points[0].y);
                
                // Рисуем контур зоны
                points.forEach((point, index) => {
                    const nextPoint = points[(index + 1) % points.length];
                    graphics.lineTo(nextPoint.x, nextPoint.y);
                });
                
                graphics.closePath();
                graphics.strokePath();
                
                // Добавляем полупрозрачную заливку
                graphics.fillStyle(GAME_CONFIG.colors.hint, 0.3);
                graphics.fillPath();
    
                // Анимация исчезновения
                this.tweens.add({
                    targets: graphics,
                    alpha: 0,
                    duration: 1000,
                    onComplete: () => {
                        graphics.destroy();
                    }
                });
            }
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
    scale: {
        mode: Phaser.Scale.FIT,
        autoCenter: Phaser.Scale.CENTER_BOTH,
        width: 1920,
        height: 1080
    },
    scene: [GameScene, InventoryScene],
    physics: {
        default: 'arcade',
        arcade: {
            debug: false
        }
    }
};

// Запуск игры
const game = new Phaser.Game(config);
