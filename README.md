<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Phaser Game</title>
    <script src="https://cdn.jsdelivr.net/npm/phaser@3.55.0/dist/phaser.min.js"></script>
    <style>
        body {
            margin: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background-color: #404040;
        }
        #renderDiv {
            width: 100%;
            height: 100%;
        }
        .button {
            position: absolute;
            padding: 15px;
            background-color: #FF5733;
            color: white;
            font-size: 20px;
            border-radius: 5px;
            border: none;
            cursor: pointer;
            display: none;
        }
        .button:hover {
            background-color: #C0392B;
        }
    </style>
</head>
<body>
    <div id="renderDiv"></div>
    <button id="restartButton" class="button">Restart</button>

    <script>
        class Card extends Phaser.GameObjects.Container {
            constructor(scene, x, y, frontTexture, backTexture, cardType, color) {
                super(scene, x, y);

                this.frontSprite = scene.add.sprite(0, 0, frontTexture);
                this.backSprite = scene.add.sprite(0, 0, backTexture);

                this.add(this.backSprite);
                this.add(this.frontSprite);

                this.setSize(this.frontSprite.width, this.frontSprite.height);
                this.setInteractive();

                this.isFlipped = false;
                this.cardType = cardType;
                this.color = color;

                // Initialize with the back face up
                this.frontSprite.setVisible(false);

                if (this.cardType === 'text') {
                    const colorName = this.getColorName(color);
                    this.text = scene.add.bitmapText(0, -30, 'roboto', colorName, 20);
                    this.text.setOrigin(0.5);
                    this.text.setVisible(false);
                    this.text.setTint(0x000000); // Set text color to black
                    this.text.setScale(1.5); // Increase the scale of the text
                    this.add(this.text);
                } else if (this.cardType === 'image') {
                    this.gamepad = scene.add.image(0, 0, 'gamepad');
                    this.gamepad.setScale(1);
                    this.gamepad.setTint(color);
                    this.gamepad.setVisible(false);
                    this.add(this.gamepad);
                }

                this.on('pointerdown', () => {
                    scene.flipCard(this);
                });

                // Add hover effect
                this.on('pointerover', this.onHoverStart, this);
                this.on('pointerout', this.onHoverEnd, this);

                // Add outline
                this.outline = scene.add.graphics();
                this.add(this.outline);
                this.outline.setVisible(false);

                // Store original position
                this.originalX = x;
                this.originalY = y;
            }

            getColorName(color) {
                const colorNames = {
                    0xFF0000: 'Red',
                    0x00FF00: 'Green',
                    0x00FFFF: 'Cyan',
                    0xFF00FF: 'Purple',
                    0xFFFF00: 'Yellow',
                    0x0000FF: 'Blue'
                };
                return colorNames[color] || 'Unknown';
            }

            flip() {
                this.scene.tweens.add({
                    targets: this,
                    scaleX: 0,
                    duration: 150,
                    onComplete: () => {
                        this.isFlipped = !this.isFlipped;
                        this.frontSprite.setVisible(this.isFlipped);
                        this.backSprite.setVisible(!this.isFlipped);
                        if (this.cardType === 'text') {
                            this.text.setVisible(this.isFlipped);
                        } else if (this.cardType === 'image') {
                            this.gamepad.setVisible(this.isFlipped);
                        }
                        this.scene.tweens.add({
                            targets: this,
                            scaleX: 1,
                            duration: 150
                        });
                    }
                });
                this.scene.sound.play('flip');
            }

            showOutline(isMatch) {
                this.outline.clear();
                this.outline.lineStyle(4, isMatch ? 0x00ff00 : 0xff0000);
                this.outline.strokeRect(-this.width / 2, -this.height / 2, this.width, this.height);
                this.outline.setVisible(true);
            }

            hideOutline() {
                this.outline.setVisible(false);
            }

            fadeOut() {
                this.scene.tweens.add({
                    targets: this,
                    alpha: 0,
                    duration: 500,
                    onComplete: () => {
                        this.setVisible(false);
                    }
                });
            }

            onHoverStart() {
                const pointer = this.scene.input.activePointer;
                const dx = this.x - pointer.x;
                const dy = this.y - pointer.y;
                const angle = Math.atan2(dy, dx);
                const distance = 10;

                this.scene.tweens.add({
                    targets: this,
                    x: this.originalX + Math.cos(angle) * distance,
                    y: this.originalY + Math.sin(angle) * distance,
                    duration: 100,
                    ease: 'Cubic.easeOut'
                });
            }

            onHoverEnd() {
                this.scene.tweens.add({
                    targets: this,
                    x: this.originalX,
                    y: this.originalY,
                    duration: 100,
                    ease: 'Cubic.easeOut'
                });
            }
        }

        class Example extends Phaser.Scene {
            constructor() {
                super();
                this.cards = [];
                this.flippedCards = [];
                this.canFlip = true;
                this.matchedPairs = 0;
                this.totalPairs = 6; // Update this if you change the number of pairs
            }

            preload() {
                this.load.image('card_front', 'https://play.rosebud.ai/assets/card_design_blank.png?8AZg');
                this.load.image('card_back', 'https://play.rosebud.ai/assets/card_design_a.png?QbCJ');
                this.load.image('gamepad', 'https://play.rosebud.ai/assets/Game-Controller.png?H276');
                this.load.audio('flip', 'https://play.rosebud.ai/assets/Retro%20Weapon%20Reload%20Best%20A%2003.wav?EG5g');
                this.load.audio('match_fail', 'https://play.rosebud.ai/assets/Retro%20Negative%20Short%2023.wav?gZhz');
                this.load.audio('match_success', 'https://play.rosebud.ai/assets/Retro%20Event%20UI%2001.wav?ob8r');
                this.load.bitmapFont('roboto', 'https://play.rosebud.ai/assets/rosebud_roboto.png?eWqX', 'https://play.rosebud.ai/assets/rosebud_roboto.xml?Jrrc');
            }

            create() {
                this.cameras.main.setBackgroundColor('#404040'); // Darker background color

                const colors = [0xFF0000, 0x00FF00, 0x00FFFF, 0xFF00FF, 0xFFFF00, 0x0000FF];
                const cardTypes = ['text', 'image'];

                const cardWidth = 120;
                const cardHeight = 180;
                const cardSpacingX = 140;
                const cardSpacingY = 200;
                const startX = (this.sys.game.config.width - (cardSpacingX * 4)) / 2 + cardWidth / 2;
                const startY = (this.sys.game.config.height - (cardSpacingY * 3)) / 2 + cardHeight / 2;

                let cardData = [];
                for (let color of colors) {
                    cardData.push({ color, type: 'text' });
                    cardData.push({ color, type: 'image' });
                }
                Phaser.Utils.Array.Shuffle(cardData);

                for (let row = 0; row < 3; row++) {
                    for (let col = 0; col < 4; col++) {
                        const x = startX + col * cardSpacingX;
                        const y = startY + row * cardSpacingY;
                        const rotation = Phaser.Math.Between(-5, 5) * (Math.PI / 180);

                        const data = cardData.pop();
                        const card = new Card(this, x, y, 'card_front', 'card_back', data.type, data.color);
                        card.setRotation(rotation);
                        this.add.existing(card);
                        this.cards.push(card);
                    }
                }

                // Set up "You Won!" text
                this.youWonText = this.add.text(this.sys.game.config.width / 2, this.sys.game.config.height / 2, 'You Won!', {
                    fontSize: '50px',
                    color: '#fff',
                    fontFamily: 'Arial'
                }).setOrigin(0.5).setVisible(false);

                // Set up Restart Button
                this.restartButton = document.getElementById('restartButton');
                this.restartButton.style.display = 'none';
                this.restartButton.addEventListener('click', () => {
                    this.scene.restart();
                });
            }

            flipCard(card) {
                if (!this.canFlip || card.isFlipped) return;

                card.flip();
                this.flippedCards.push(card);

                if (this.flippedCards.length === 2) {
                    this.canFlip = false;
                    this.checkMatch();
                }
            }

            checkMatch() {
                const [card1, card2] = this.flippedCards;
                const isMatch = card1.color === card2.color && card1.cardType !== card2.cardType;

                card1.showOutline(isMatch);
                card2.showOutline(isMatch);

                if (isMatch) {
                    this.sound.play('match_success');
                    this.shakeScreen();
                    this.time.delayedCall(2000, () => {
                        card1.fadeOut();
                        card2.fadeOut();
                        this.matchedPairs++;
                        this.resetFlippedCards();

                        if (this.matchedPairs === this.totalPairs) {
                            this.youWonText.setVisible(true);
                            this.restartButton.style.display = 'block'; // Show the Restart Button
                        }
                    });
                } else {
                    this.sound.play('match_fail');
                    this.time.delayedCall(2000, () => {
                        card1.flip();
                        card2.flip();
                        card1.hideOutline();
                        card2.hideOutline();
                        this.resetFlippedCards();
                    });
                }
            }

            resetFlippedCards() {
                this.flippedCards = [];
                this.canFlip = true;
            }

            shakeScreen() {
                this.cameras.main.shake(250, 0.005);
            }
        }

        const config = {
            type: Phaser.AUTO,
            parent: 'renderDiv',
            pixelArt: false,
            scale: {
                mode: Phaser.Scale.FIT,
                autoCenter: Phaser.Scale.CENTER_BOTH,
            },
            width: 1000,
            height: 800,
            scene: Example,
            antialias: true,
            roundPixels: false,
        };

        window.phaserGame = new Phaser.Game(config);
    </script>
</body>
</html>
