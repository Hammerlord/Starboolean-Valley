int canvasWidth = 500;
int canvasHeight = 600;

StarBooleanValley game;

void setup() { 
    size(500, 600); // The previewer can't read variables but this should be same as canvas width and height
    noStroke();
    smooth();
    game = new StarBooleanValley();
    game.start();
} 

void draw() {
    game.update();
}

void mouseMoved() {
    game.moveBasket(mouseX);
}

final class StarBooleanValley {
    private final SpawnTimer appleSpawnTimer;
    private ArrayList<Apple> apples;
    private Tree tree;
    private final float gravity = 1.25;
    private final Basket basket;
    private final Scenery scenery;
    private int applesCaught;
    private int applesFell;
    private TimeTracker timeTracker;
    
    StarBooleanValley() {
        this.timeTracker = new TimeTracker();
        this.scenery = new Scenery(canvasWidth, canvasHeight, timeTracker);
        this.applesCaught = 0;
        this.applesFell = 0;
        this.appleSpawnTimer = new SpawnTimer();
        this.apples = new ArrayList<Apple>();
        this.tree = new Tree(canvasWidth / 2, canvasHeight - 400); // TODO
        this.basket = new Basket(new Position(0, canvasHeight - Basket.height - 10));
        this.start();
    }
    
    void start() {
        this.appleSpawnTimer.start();
        this.render();
    }
    
    void update() {
        this.timeTracker.update();
        this.scenery.update();
        if (this.appleSpawnTimer.checkSpawn(this.getMultiplier(2.0))) {
            this.apples.add(this.spawnApple());
        }
        for (int i = this.apples.size() - 1; i >= 0; --i) {
            Apple apple = this.apples.get(i);
            apple.update();
            if (this.basket.didCatch(apple)) {
                ++this.applesCaught;
                this.apples.remove(apple);
            } else if (this.didFall(apple)) {
                ++this.applesFell;
                this.apples.remove(apple);
            }
        }
        this.render();
    }
    
    void render() {
        this.scenery.render();
        float timeOfDay = this.timeTracker.timeOfDay();
        this.tree.render(timeOfDay);
        for (int i = this.apples.size() - 1; i >= 0; --i) {
            Apple apple = this.apples.get(i);
            apple.render(timeOfDay);
        }
        this.basket.render();
        this.renderScore();
    }
    
    void moveBasket(mouseX) {
        this.basket.setX(mouseX);
    }
    
    private Apple spawnApple() {
        return new Apple(
            this.tree.getAppleSpawnPosition(), 
            this.gravity,
            this.getMultiplier(2.0)
        );
    }
    
    private boolean didFall(Apple apple) {
        return apple.getY() > canvasHeight;
    }
    
    private void renderScore() {
        fill(255);
        text("Apples caught: " + this.applesCaught, canvasWidth - 100, 20);
        text("Apples fallen: " + this.applesFell, canvasWidth - 100, 40);
    }
    
    private float getMultiplier(float maxMultiplier) {
        return min(maxMultiplier, 1 + this.applesCaught / 100);
    }
}


final class TimeTracker {
    private int time;
    private int pendulum;
    private final int transitionTime;
    
    TimeTracker() {
        this.transitionTime = 60000;
        this.pendulum = -1;
        this.time = this.transitionTime;
    }
    
    void update() {
        this.time += 60 * this.pendulum;
        if (abs(this.time) - this.transitionTime >= 0) {
            this.pendulum *= -1;
        }
    }
    
    float timeOfDay() {
        // Represented by a percentage. Closer to 0%: nighttime; closer to 100%: daytime.
        return (this.time + this.transitionTime) / (this.transitionTime * 2);
    }
}


final class Scenery {
    private final Sun sun;
    private final Moon moon;
    private final int width;
    private final int height;
    private final TimeTracker timeTracker;
    private final ArrayList<Star> stars;
    
    Scenery(int width, int height, TimeTracker timeTracker) {
        this.width = width;
        this.height = height;
        this.timeTracker = timeTracker;
        this.sun = new Sun(new Position(100, 0));
        this.moon = new Moon(new Position(width - 100, 0));
        this.stars = new ArrayList<Star>();
        for (int i = 0; i < random(5, 7); ++i) {
            Star star = new Star(new Position(random(0, width), random(0, height * 0.25)));
            this.stars.add(star);
        }
        this.render();
    }
    
    void update() {
        int maximum = this.height + 300; // TODO out of sight
        float timeOfDay = this.timeTracker.timeOfDay();
        float sunY = max(75, maximum - maximum * timeOfDay);
        this.sun.setY(sunY);
        float moonY = max(75, maximum * timeOfDay);
        this.moon.setY(moonY);
    }
    
    void render() {
        this.renderSky();       
        float timeOfDay = this.timeTracker.timeOfDay();
        for (int i = 0; i < this.stars.size(); ++i) {
            this.stars.get(i).render(timeOfDay);
        }
        this.sun.render(timeOfDay);
        this.moon.render(timeOfDay);
        this.renderGrass();
    }
    
    private void renderSky() {
        float timeOfDay = this.timeTracker.timeOfDay();
        background(this.adjustBrightness(50, timeOfDay),
                   this.adjustBrightness(180, timeOfDay),
                   this.adjustBrightness(255, timeOfDay));
    }
    
    private void renderGrass() {
        float timeOfDay = this.timeTracker.timeOfDay();
        fill(this.adjustBrightness(40, timeOfDay),
             this.adjustBrightness(200, timeOfDay),
             this.adjustBrightness(100, timeOfDay));
        rect(0, this.height - 50, this.width, this.height);
    }
    
    private int adjustBrightness(int colorValue, timeOfDay) {
        return int(colorValue * 0.75 + ((colorValue * 0.25) * timeOfDay));
    }
}


final class Apple {
    final static int radius = 20;
    final static int diameter = Apple.radius * 2;
    private final Position position;
    private final int launchTime;
    private final int spawnTime;
    private final float gravity;
    private final float launchForceMultiplier;
    
    private float rotation;
    private float movementForceY;
    private float movementForceX;
    
    private boolean didLaunch;
    
    Apple(Position position, float gravity, float launchForceMultiplier) {
        this.position = position;
        this.rotation = 0;
        this.launchTime = random(2500, 4000);
        this.spawnTime = millis();
        this.didLaunch = false;
        this.gravity = gravity;
        this.movementForceY = 0;
        this.movementForceX = 0;
        this.launchForceMultiplier = launchForceMultiplier || 1;
    }
    
    float getX() {
        return this.position.x;
    }
    
    float getY() {
        return this.position.y;
    }
    
    void render(float timeOfDay) {
        pushMatrix();
        translate(this.position.x, this.position.y);
        rotate(radians(this.rotation));
        this.renderBody(timeOfDay);
        this.renderStem(timeOfDay);
        popMatrix();
    }
    
    void update() {
        if (this.didLaunch) {
            this.updateRotation();
            this.checkHitWall();
            this.position.translate(this.movementForceX, this.movementForceY);
            this.movementForceY += this.gravity;
        } else {
            this.checkLaunch();
        }
    }
    
    private void renderBody(float timeOfDay) {
        fill(this.adjustBrightness(245, timeOfDay),
             this.adjustBrightness(55, timeOfDay),
             this.adjustBrightness(45, timeOfDay));
        ellipse(0, 0, Apple.diameter, Apple.diameter);
        float halfRadius = Apple.radius / 2;
        ellipse(4 - halfRadius, halfRadius + 1, Apple.radius, Apple.radius);
        ellipse(halfRadius - 4, halfRadius + 1, Apple.radius, Apple.radius);
    }
    
    private void renderStem(float timeOfDay) {
        beginShape();
        fill(this.adjustBrightness(70, timeOfDay), 
             this.adjustBrightness(35, timeOfDay),
             this.adjustBrightness(10, timeOfDay));
        vertex(0, 0 - Apple.radius * 0.5);
        bezierVertex(0, -Apple.radius * 0.5, 
                     -5, -Apple.radius, 
                     5, -Apple.radius * 1.25);
        endShape();
    }
    
    private void checkLaunch() {
        if (millis() - spawnTime >= this.launchTime) {
            this.didLaunch = true;
            int maxX = 10 * this.launchForceMultiplier;
            this.movementForceX = random(-maxX, maxX);
            this.movementForceY = random(-7, -15);
        }
    }
    
    private void checkHitWall() {
        if (this.position.x - Apple.radius < 0 || 
            this.position.x + Apple.radius > canvasWidth) {
            this.movementForceX = -this.movementForceX / 2;
        }
    }
    
    private void updateRotation() {
        this.rotation += this.movementForceX;
        if (rotation > 360) {
            rotation = 0;
        } else if (rotation < 0) {
            rotation = 360;
        }
    }
    
    private int adjustBrightness(int colorValue, float timeOfDay) {
        float half = colorValue * 0.5;
        return int(half + (half * timeOfDay));
    }
}


final class Moon {
    private final Position position;
    static final int diametre = 80;
    
    Moon(Position position) {
        this.position = position;
    }
    
    void render(float timeOfDay) {
        for (let i = 0; i < 3; ++i) {
            int diametre = Moon.diametre + i * 20;
            int baseAlpha = 200 - i * 10;
            fill(220, 220, 175, baseAlpha * (1 - timeOfDay));
            ellipse(this.position.x, this.position.y, diametre, diametre);
        }
    }
    
    void setY(float y) {
        this.position.setY(y);
    }
}


final class Star {
    private final Position position;
    static final int diametre = 10;
    
    Star(Position position) {
        this.position = position;
    }
    
    void render(float timeOfDay) {
        for (let i = 0; i < 2; ++i) {
            int diametre = Star.diametre + i * 10;
            int baseAlpha = 200 - i * 20;
            fill(255, 255, 220, baseAlpha * (1 - timeOfDay));
            ellipse(this.position.x, this.position.y, diametre, diametre);
        }
    }
}


final class Sun {
    final Position position;
    final int radius = 50;
    
    Sun(Position position) {
        this.position = position;
    }
    
    void render(float timeOfDay) {
        int diameter = this.radius * 2;
        fill(color(255, 255, 0, timeOfDay * 2 * 255));
        ellipse(this.position.x, this.position.y, diameter, diameter);
        this.renderPointy(timeOfDay);
        this.renderEyes();
        this.renderMouth();
    }
    
    void setY(float y) {
        this.position.setY(y);
    }
    
    private void renderEyes() {
        fill(color(0, 0, 0));
        int offset = this.radius / 2;
        this.renderEye(this.position.x - offset);
        this.renderEye(this.position.x + offset);     
    }
    
    private void renderEye(int x) {
        ellipse(x, this.position.y - 10, 10, 10);
    }
    
    private void renderMouth() {
        fill(color(255, 0, 0));
        arc(this.position.x, this.position.y + 5, 60, 55, 0, PI);
    }
    
    private void renderPointy() {
        int offset = 7;
        int radiusOffset = this.radius * 1.75;
        int x = this.position.x;
        int y = this.position.y;
        triangle(x - radiusOffset, y,
                 x - radius + 1, y - offset,
                 x - radius + 1, y + offset); // Left
        triangle(x + radiusOffset, y, 
                 x + radius - 1, y - offset, 
                 x + radius - 1, y + offset); // Right
        triangle(x, y - radiusOffset, 
                 x - offset, y - radius + 1, 
                 x + offset, y - radius + 1); // Top
        triangle(x, y + radiusOffset, 
                 x - offset, y + radius - 1, 
                 x + offset, y + radius - 1); // Bottom
    }
}


final class SpawnTimer {
    private int millisToNext;
    private int lastSpawnTime;
    private boolean isActive;
    private final int spawnSpeed;
    
    SpawnTimer() {
        this.millisToNext = 0;
        this.isActive = false;
        this.spawnSpeed = 2000; // millis
    }
    
    void start() {
        this.isActive = true;
        this.lastSpawnTime = millis();
    }
    
    boolean checkSpawn(float speedMultiplier) {
        if (!this.isActive) {
            return false;
        }
        if (millis() - lastSpawnTime >= this.millisToNext) {
            this.onTrigger(speedMultiplier);
            return true;
        }
        return false;
    }
    
    private void onTrigger(float speedMultiplier) {
        int spawnSpeed = this.spawnSpeed - (this.spawnSpeed * speedMultiplier - this.spawnSpeed);
        int fastest = spawnSpeed - spawnSpeed / 2;
        int slowest = spawnSpeed + 1000;
        this.lastSpawnTime = millis();
        this.millisToNext = int(random(fastest, slowest)); // TODO
    }
}


final class Tree {
    final static int bushRadius = 70;
    final static int bushDiametre = Tree.bushRadius * 2;
    private final Position position;
    private final Position[] bushPositions;
    
    Tree(int x, int y) {
        int diagonalOffset = Tree.bushRadius / 1.5;
        this.position = new Position(x, y);
        this.bushPositions = {
            this.position,
            new Position(x - Tree.bushRadius, y),
            new Position(x + Tree.bushRadius, y),
            new Position(x, y - Tree.bushRadius),
            new Position(x, y + Tree.bushRadius),
            new Position(x - diagonalOffset, y - diagonalOffset),
            new Position(x + diagonalOffset, y - diagonalOffset),
            new Position(x + diagonalOffset, y + diagonalOffset),
            new Position(x - diagonalOffset, y + diagonalOffset)
        };
    }
    
    void render(float timeOfDay) {
        this.renderTrunk(timeOfDay);
        this.renderBushes(timeOfDay);
        this.renderFace(timeOfDay);
    }
    
    Position getAppleSpawnPosition() {
        int randomBushPosition = int(random(this.bushPositions.length));
        Position pos = this.bushPositions[randomBushPosition];
        float x = random(pos.x - Tree.bushRadius, pos.x + Tree.bushRadius);
        float y = random(pos.y - Tree.bushRadius, pos.y + Tree.bushRadius);
        // TODO it's a square.
        return new Position(x, y);
    }
    
    private void renderTrunk(float timeOfDay) {
        fill(this.adjustBrightness(100, timeOfDay),
             this.adjustBrightness(70, timeOfDay),
             this.adjustBrightness(50, timeOfDay));
        int x = this.position.x;
        int y = this.position.y;
        int startY = this.position.y + Tree.bushRadius;
        int width = 30;
        rect(x - width / 2, startY, width, startY + Tree.bushRadius);
    }
    
    private void renderBushes(float timeOfDay) {
        fill(this.adjustBrightness(50, timeOfDay),
             this.adjustBrightness(230, timeOfDay),
             this.adjustBrightness(120, timeOfDay));
        for (let i = 0; i < this.bushPositions.length; ++i) {
            Position pos = this.bushPositions[i]; 
            this.renderBush(pos.x, pos.y);
        }
    }
    
    private void renderBush(int x, int y) {
        ellipse(x, y, Tree.bushDiametre, Tree.bushDiametre);
    }
    
    private void renderFace(float timeOfDay) {
        fill(this.adjustBrightness(20, timeOfDay),
             this.adjustBrightness(100, timeOfDay),
             this.adjustBrightness(50, timeOfDay));
        int x = this.position.x;
        int y = this.position.y;
        int eyeWidth = 30;
        int eyeHeight = 50;
        ellipse(x - Tree.bushRadius / 2 - eyeWidth / 2, y - eyeHeight / 2, eyeWidth, eyeHeight);
        ellipse(x + Tree.bushRadius - eyeWidth / 2, y - eyeHeight / 2, eyeWidth, eyeHeight);
        arc(x, y + Tree.bushRadius / 2, 100, 70, 0, PI);
    }
    
    private int adjustBrightness(int colorValue, timeOfDay) {
        return int(colorValue * 0.75 + ((colorValue * 0.25) * timeOfDay));
    }
}


final class Basket {
    private Position position;
    final static int width = 120;
    final static int height = 80;
    
    Basket(Position position) {
        this.position = position;
    }
    
    void render() {
        fill(color(120, 90, 70));
        float x = this.position.x;
        float y = this.position.y;
        int width = Basket.width;
        height = Basket.height;
        quad(x - width * 0.1, y,
             x + width + width * 0.1, y,
             x + width, y + height,
             x, y + height);
    }
    
    void setX(float x) {
        this.position.setX(x);
        if (this.position.x > canvasWidth - Basket.width) {
            this.position.setX(canvasWidth - Basket.width);
        } else if (this.position.x < 0) {
            this.position.setX(0);
        }
    }
    
    boolean didCatch(Apple apple) {
        return apple.getX() > this.position.x && 
               apple.getX() < this.position.x + Basket.width &&
               apple.getY() > this.position.y;
    }
}


final class Position {
    float x;
    float y;
    
    Position(int x, int y) {
        this.x = x || 0;
        this.y = y || 0;
    }
    
    void setX(int x) {
        this.x = x;
    }
    
    void setY(int y) {
        this.y = y;
    }
    
    void translateX(float amount) {
        this.x += amount;
    }
    
    void translateY(float amount) {
        this.y += amount;
    }
    
    void translate(float x, float y) {
        this.translateX(x);
        this.translateY(y);
    }
}

