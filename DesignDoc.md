# Plants vs. Zombies Design Document (C++)

## Core Game Logic

### `GameWorld` Class

The `GameWorld` class is central to managing the game state. It needs a container, `std::list<std::shared_ptr<GameObject>>`, to store all game objects. In each game tick (a discrete unit of game time), the `GameWorld` must iterate through this list and call the `Update()` method on every object.

The `CleanUp()` function is the cleanup step at the end of a level. When the `Update()` function of `GameWorld` returns a status indicating the current level has ended, the game framework will call this function. You need to clear all game objects from the current level, ensuring no memory leaks occur.

Your `GameWorld` needs to store all game objects in a single container. Therefore, only by having all game objects inherit from the same base class can polymorphism be utilized to "perform differentiated operations on each object without needing to know its specific type."

### `GameObject` Class

`GameObject` inherits from `ObjectBase`, a lower-level base class provided by the game framework. It also inherits from `std::enable_shared_from_this`, allowing the use of `shared_from_this()` to create a smart pointer to itself, replacing the raw `this` pointer. This is used for clearly separating the tasks of true objects from those handled by the game framework, like how to display plant and zombie animations.

#### What `GameObject` Should and Shouldn't Store

`GameObject` inherits five basic attributes from `ObjectBase` class, such as the x and y coordinates of its position. You do not need to store these basic attributes again in your `GameObject`; instead, you should call the functions provided in `ObjectBase` to get/modify them.

Besides the basic attributes defined in `ObjectBase` listed below, if you believe there are other attributes that all game objects must have, you should define them in your `GameObject` class. If some attributes are specific to plants but not zombies, define them in the plant class rather than cramming everything into `GameObject`.

#### Introduction to Inherited Basic Attributes

Basic attributes in `ObjectBase` include:

*   **`imageID`**: Represents the texture material ID for this object. All IDs are defined in "utils.hpp".
    *   `imageID` must be determined when the object is created. It only needs to be modified using `ChangeImage(ImageID imageID)` when a Wall-nut or Buckethead Zombie changes its texture.
    *   Since `imageID` corresponds to the actual type of each object, and the idea of polymorphism should not use information about an object's actual type, `imageID` does not provide an accessor function. When writing this game, ideas like "wanting to know the object's `imageID`" or "knowing what specific thing it is" are incorrect. You should use virtual functions with differentiated implementations to determine which broad categories certain objects belong to.
*   **`x` and `y`**: Represent the object's current coordinates in pixels. The screen's bottom-left corner is the origin (0, 0), with the x-axis positive to the right and the y-axis positive upwards. The top-right corner of the screen is at (`WINDOW_WIDTH` – 1, `WINDOW_HEIGHT` – 1). Accessor functions `GetX()` and `GetY()`, and a modification function `MoveTo(int x, int y)` to change both x and y simultaneously are provided.
*   **`layer`**: Represents the object's display layer on the screen, with a value range of [0, `MAX_LAYER`]. Objects with lower layer values will cover objects with higher layer values when displayed. For example, Sun is at layer 0 while Sunflower is at layer 3, so the Sun will cover the Sunflower when displayed.
*   **`width` and `height`**: Represent the object's (collision box) size, type `int`. Accessor functions `GetWidth()`, `GetHeight()`, and modification functions `SetWidth(int width)`, `SetHeight(int height)` are provided.
*   **`AnimID`**: Represents the animation this object has. Accessor function `GetCurrentAnimation()` is provided. When an object needs to change its animation, use `PlayAnimation(AnimID animID)` to modify it, starting the new animation from the first frame.

The above attributes are also parameters required by the `ObjectBase` constructor. Because `ObjectBase` does not allow default construction, in the constructors of your `GameObject` and other subclasses, you need to provide these parameters to the base class `ObjectBase` using an initializer list.

*   The `GameObject` class can similarly require all classes inheriting from it to provide an `imageID`. Besides `imageID`, other undetermined parts like `x` and `y` can also be placed in the abstract class's constructor and passed down to concrete subclasses. When a subclass has definite attribute values (e.g., Sun has a definite `imageID`), these are passed back up through successive calls to base class constructors, ultimately providing them to the lowest-level `ObjectBase`.

Furthermore, `ObjectBase` does not allow copy/move construction or copy/move assignment. To explain intuitively, the right to manage game objects should belong solely to `GameWorld`, and no object should be able to easily copy itself or other objects.

#### Representing Each Object Type by Inheriting `GameObject`

Once your `GameObject` class inherits from `ObjectBase`, you can continue to create subclasses for each specific type of object, defining their behaviors. Each type of game object can implement the `Update()` function differently. In the `Update()` function, your game object can move, change its state, or even "die."

Objects like Sun that are not clicked within a certain time, zombies that are killed, or plants that are shoveled need to "die." Note that all objects in the game are managed by the container in `GameWorld`; no object, including itself, has the right to clean up objects. Therefore, an object "dies" by marking itself as "dead" or by being judged as such (e.g., HP is zero). After all objects have performed one `Update()`, `GameWorld` can collectively clean up all objects marked as "dead," i.e., remove them from the container.

## `GameWorld` Implementation Details

### `GameWorld::Init()`

1.  Initialize any member variables used for recording level data. For example, initial wave number is 0, initial sun is 50.
2.  Create text displays for level data. You need to use the `TextBase` class, which is used similarly to all `GameObject`s you create. 
    *   Sun count is recommended to be displayed at (60, `WINDOW_HEIGHT` - 78).
    *   Wave number is recommended to be displayed at (`WINDOW_WIDTH` - 160, 8).
3.  Create the background.
4.  Create 45 planting spots, `GAME_ROWS` (5) rows and `GAME_COLS` (9) columns
    *   The first (bottom-left) spot is at (x = `FIRST_COL_CENTER`, y = `FIRST_ROW_CENTER`).
    *   The horizontal spacing between planting spots is `LAWN_GRID_WIDTH`, and the vertical spacing is `LAWN_GRID_HEIGHT`.
5.  Generate plant seed buttons at the top:
    *   The first button is the Sunflower seed, located at (x = 130, y = `WINDOW_HEIGHT` – 44).
    *   The remaining buttons, in order, are Peashooter, Wall-nut, Cherry Bomb, Repeater, with a horizontal spacing of 60.
6.  Generate the shovel button, located at (x = 600, y = `WINDOW_HEIGHT` – 40).

### `GameWorld::Update()`

1.  **Drop new suns for the game.** The first sun drops at the 180th tick (6 seconds) after the game starts. Thereafter, a sun drops every 300 ticks (10 seconds). The generated sun:
    *   Has an x-coordinate that is a random `int` in the range [75, `WINDOW_WIDTH` - 75].
    *   The function `randInt()` for generating a random integer in a closed interval is provided in `utils.hpp`.
    *   Has a y-coordinate of `WINDOW_HEIGHT` – 1.
    *   Will start falling.
2.  **Generate new zombies for the game.** Zombies appear in waves. The first wave of zombies will generate at the 1200th tick (40 seconds) after the game starts. The generation interval for subsequent waves will decrease. The specific strategy is as follows:
    *   Let the current zombie wave number be `wave`. (Game starts with `wave` = 0, first zombie wave is `wave` = 1).
    *   Randomly generate ⌊((15 + `wave`)) / 10⌋ zombies.
    *   The next wave of zombies will generate after `max(150, 600 – 20 * wave)` ticks.
3.  **If zombies are determined to be generated in the previous step, further randomly decide the type of each zombie.** For each generated zombie, the strategy is as follows:
    *   Let P1 = 20,
        P2 = 2 * `max(wave – 8, 0)`,
        P3 = 3 * `max(wave – 15, 0)`. Then:
    *   With probability P1 / (P1 + P2 + P3), the generated zombie is a Regular Zombie.
    *   With probability P2 / (P1 + P2 + P3), the generated zombie is a Pole Vaulting Zombie.
    *   With the remaining probability P3 / (P1 + P2 + P3), the generated zombie is a Buckethead Zombie.
    *   The generated zombie's x-coordinate is a random `int` in the range [`WINDOW_WIDTH` - 40, `WINDOW_WIDTH` – 1], and its y-coordinate is the y-coordinate of a random row (the first row is `FIRST_ROW_CENTER`, vertical spacing is `LAWN_GRID_HEIGHT`).
4.  Iterate through all game objects (`GameObject`) and call their `Update()` functions sequentially.
5.  **Detect collisions.** Be careful not to trigger the same collision multiple times. Possible collisions include:
    *   Pea hits a zombie.
    *   Explosion affects a zombie.
    *   Zombie bites a plant.
    *   Special collision detection for Pole Vaulting Zombie will be done within its `Update()`.
6.  Iterate through all game objects again, removing objects that need to be deleted from your storage container.
    *   Objects to be deleted should be marked as "dead" or "HP is zero" so `GameWorld` can find them.
7.  **Determine if the game is lost.** If any zombie's x-coordinate is < 0, return `LevelStatus::LOSING`.
    *   On the failure screen, a space is reserved for displaying the number of game waves you completed. You can create a text object at this time to display your game result. It is recommended to create it in white (RGB = 1, 1, 1) at position (330, 50).
8.  **Perform an additional check to see if zombies are colliding with plants.** If a zombie is not colliding with a plant but is in an eating state, make it walk normally.
    *   This situation occurs when multiple zombies kill the same plant in the same frame. The zombies that started eating earlier do not know the plant will die and can remain in the eating state, so an extra check is needed to make them move normally.
9.  Update the content of the text displays (`TextBase`) you created in the game to correctly show necessary information, such as sun count, wave number, etc.
10. Return `LevelStatus::ONGOING`, indicating the current level is running normally.

### `GameWorld::CleanUp()`

Your `GameWorld::CleanUp()` function must clear the container you use. If you have saved other objects, they also need to be cleared correctly.

## Game Object Specifications

### Background

*   **When created:**
    *   Image ID: `IMGID_BACKGROUND`.
    *   Position: (x = `WINDOW_WIDTH` / 2, y = `WINDOW_HEIGHT` / 2).
    *   Layer: `LAYER_BACKGROUND`.
    *   Width: `WINDOW_WIDTH`, Height: `WINDOW_HEIGHT`.
    *   Animation: None (Animation ID: `ANIMID_NO_ANIMATION`).
*   **When `Update()` is called:**
    *   Does nothing.
*   **When clicked:**
    *   Does nothing.

### Sun

*   **When created:**
    *   Image ID: `IMGID_SUN`.
    *   Starting position: Not fixed, determined by the code that creates it. (Therefore, its constructor must include this part.)
    *   Layer: `LAYER_SUN`.
    *   Width: 80, Height: 80.
    *   Animation: `ANIMID_IDLE_ANIM`.
    *   Has a fall duration.
        *   Sun falling from the sky has a fall duration of a random `int` between [63, 263]. This duration ensures it lands on the lawn.
        *   Sun produced by a Sunflower will land in a parabola, with a fall duration of 12 ticks.
*   **When `Update()` is called:**
    *   If the sun has not yet landed:
        *   If it's a sun falling from the sky, it moves down 2 pixels per tick.
        *   If it's a sun produced by a Sunflower, it moves left 1 pixel per tick and undergoes vertical projectile motion: its initial vertical velocity (first frame) is 4 pixels/tick upwards, with an acceleration of -1 (downwards).
        *   This sun's trajectory is a parabola moving upwards and to the left. At the 5th tick, its y-direction velocity is 0, at the apex of the parabola.
    *   If the sun has landed, it will no longer move and will disappear after 300 ticks (10 seconds) on the ground.
*   **When clicked:**
    *   When clicked, the sun needs to disappear and notify `GameWorld` to gain 25 sun points.

### Planting Spot

Planting Spots are "invisible rabbits" on the lawn. Clicking one plants the held plant at its location.

*   **When created:**
    *   Image ID: `IMGID_NONE` (no texture).
    *   Position: Not fixed, determined by the code that creates it. (Therefore, its constructor must include this part.)
    *   Layer: `LAYER_UI`.
    *   Width: 60, Height: 80. Slightly smaller than a lawn grid, consistent with plant size.
    *   Animation: None (Animation ID: `ANIMID_NO_ANIMATION`).
*   **When `Update()` is called:**
    *   Does nothing.
*   **When clicked:**
    *   If the player is holding an unplanted seed, "plant" it, i.e., generate the corresponding plant at the planting spot's location.
    *   The planted plant, being on a more forward layer, will cover the planting spot. Thus, a planting spot with a plant will no longer be clickable.

### Sunflower Seed

*   **When created:**
    *   Image ID: `IMGID_SEED_SUNFLOWER`.
    *   Position: First seed packet location (x = 130, y = `WINDOW_HEIGHT` - 44).
    *   Layer: `LAYER_UI`.
    *   Width: 50, Height: 70.
    *   Animation: None.
    *   Price: 50 sun.
    *   Cooldown time: 240 ticks (8 seconds).
*   **When `Update()` is called:**
    *   The Sunflower Seed's `Update()` doesn't necessarily have to do anything, but based on your implementation, you can do something in `Update()`...
*   **When clicked:**
    *   If the player is holding a shovel or an unplanted seed, the click is invalid.
    *   If the Sunflower Seed is on cooldown, the click is invalid. (If you implement a cooldown mask, it will block the click).
    *   Ask `GameWorld` if there are 50 sun points. If yes, spend 50 sun, enter cooldown state, generate a Cooldown Mask at its own (x, y) position, and plant a Sunflower on the next click on a Planting Spot.
    *   After spending sun, you cannot directly generate a Sunflower, because only after clicking a Planting Spot will you know where it should be generated.

### Sunflower

*   **When created:**
    *   Image ID: `IMGID_SUNFLOWER`.
    *   Position: Not fixed, determined by the code that creates it.
    *   Layer: `LAYER_PLANTS`.
    *   Width: 60, Height: 80.
    *   Animation: `ANIMID_IDLE_ANIM`.
    *   HP: 300.
    *   First sun production time: Random `int` between [30, 600] ticks. Subsequent sun production interval: 600 ticks.
*   **When `Update()` is called:**
    *   The Sunflower must first check if it is dead. If dead, it will wait for `GameWorld` to clean it up. Its `Update()` should return immediately, ignoring subsequent steps.
    *   If the Sunflower produces sun at this tick, it will generate a "sun produced by Sunflower" at its own (x, y) position.
*   **When clicked:**
    *   If the player is holding a shovel, the Sunflower needs to die. Put down the shovel.

### Cooldown Mask

*   **When created:**
    *   Image ID: `IMGID_COOLDOWN_MASK`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_COOLDOWN_MASK` (will cover the seed).
    *   Width: 50, Height: 70.
    *   Animation: None.
    *   Needs to disappear after the corresponding seed's cooldown finishes.
*   **When `Update()` is called:**
    *   Needs to disappear after the corresponding seed's cooldown finishes.
*   **When clicked:**
    *   Does nothing.

### Peashooter Seed

*   **When created:**
    *   Image ID: `IMGID_SEED_PEASHOOTER`.
    *   Position: 2nd seed packet location, horizontal spacing between seed packets is 60.
    *   Layer: `LAYER_UI`.
    *   Width: 50, Height: 70.
    *   Animation: None.
    *   Price: 100 sun.
    *   Cooldown time: 240 ticks (8 seconds).
*   **When `Update()` is called:**
    *   The Peashooter Seed's `Update()` doesn't necessarily have to do anything...
*   **When clicked:**
    *   If the player is holding a shovel or an unplanted seed, the click is invalid.
    *   If the Peashooter Seed is on cooldown, the click is invalid.
    *   Ask `GameWorld` if there are 100 sun points. If yes, spend 100 sun, enter cooldown, generate a Cooldown Mask at its position, and plant a Peashooter on the next click on a Planting Spot.

### Peashooter

*   **When created:**
    *   Image ID: `IMGID_PEASHOOTER`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_PLANTS`.
    *   Width: 60, Height: 80.
    *   Animation: `ANIMID_IDLE_ANIM`.
    *   HP: 300.
    *   Fires a pea every 30 ticks (1 second).
*   **When `Update()` is called:**
    *   Must first check if it is dead. If dead, wait for `GameWorld` cleanup. `Update()` should return immediately.
    *   If its attack ability is on cooldown, decrease cooldown by 1, `Update()` returns.
    *   If it can attack, it will ask `GameWorld` if there is a zombie to its right in its row. If yes, generate a Pea 30 pixels to its right and 20 pixels above its own coordinates, then enter a 30-tick cooldown.
*   **When clicked:**
    *   If the player is holding a shovel, the Peashooter needs to die. Put down the shovel.

### Pea

*   **When created:**
    *   Image ID: `IMGID_PEA`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_PROJECTILES`.
    *   Width: 28, Height: 28.
    *   Animation: None.
*   **When `Update()` is called:**
    *   Must first check if it is dead. If dead, wait for `GameWorld` cleanup. `Update()` should return immediately.
    *   Moves right 8 pixels.
    *   If it flies off the right screen edge (x >= `WINDOW_WIDTH`), it needs to die.
*   **When clicked:**
    *   Does nothing.
*   **When colliding with a zombie:**
    *   Deals 20 damage to the zombie it collides with, then dies.

### Wall-nut Seed

*   **When created:**
    *   Image ID: `IMGID_SEED_WALLNUT`.
    *   Position: 3rd seed packet location, horizontal spacing 60.
    *   Layer: `LAYER_UI`.
    *   Width: 50, Height: 70.
    *   Animation: None.
    *   Price: 50 sun.
    *   Cooldown time: 900 ticks (30 seconds).
*   **When `Update()` is called:**
    *   The Wall-nut Seed's `Update()` doesn't necessarily have to do anything...
*   **When clicked:**
    *   If the player is holding a shovel or an unplanted seed, the click is invalid.
    *   If the Wall-nut Seed is on cooldown, the click is invalid.
    *   Ask `GameWorld` if there are 50 sun points. If yes, spend 50 sun, enter cooldown, generate a Cooldown Mask, and plant a Wall-nut on the next click on a Planting Spot.

### Wall-nut

*   **When created:**
    *   Image ID: `IMGID_WALLNUT`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_PLANTS`.
    *   Width: 60, Height: 80.
    *   Animation: `ANIMID_IDLE_ANIM`.
    *   HP: 4000.
*   **When `Update()` is called:**
    *   Must first check if it is dead. If dead, wait for `GameWorld` cleanup. `Update()` should return immediately.
    *   If its HP is less than 1/3 of total HP, change its texture to "damaged Wall-nut" `IMGID_WALLNUT_CRACKED`.
*   **When clicked:**
    *   If the player is holding a shovel, the Wall-nut needs to die. Put down the shovel.

### Cherry Bomb Seed

*   **When created:**
    *   Image ID: `IMGID_SEED_CHERRY_BOMB`.
    *   Position: 4th seed packet location, horizontal spacing 60.
    *   Layer: `LAYER_UI`.
    *   Width: 50, Height: 70.
    *   Animation: None.
    *   Price: 150 sun.
    *   Cooldown time: 1200 ticks (40 seconds).
*   **When `Update()` is called:**
    *   The Cherry Bomb Seed's `Update()` doesn't necessarily have to do anything...
*   **When clicked:**
    *   If the player is holding a shovel or an unplanted seed, the click is invalid.
    *   If the Cherry Bomb Seed is on cooldown, the click is invalid.
    *   Ask `GameWorld` if there are 150 sun points. If yes, spend 150 sun, enter cooldown, generate a Cooldown Mask, and plant a Cherry Bomb on the next click on a Planting Spot.

### Cherry Bomb

*   **When created:**
    *   Image ID: `IMGID_CHERRY_BOMB`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_PLANTS`.
    *   Width: 60, Height: 80.
    *   Animation: `ANIMID_IDLE_ANIM`.
    *   HP: 4000 (to ensure it's not eaten by zombies before exploding).
*   **When `Update()` is called:**
    *   Needs to die on the 15th frame after being planted and generate an explosion effect at its location.
*   **When clicked:**
    *   If the player is holding a shovel, the Cherry Bomb needs to die. Put down the shovel.

### Explosion

*   **When created:**
    *   Image ID: `IMGID_EXPLOSION`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_PROJECTILES`.
    *   Width: 3 * `LAWN_GRID_WIDTH`, Height: 3 * `LAWN_GRID_HEIGHT`.
    *   Animation: None.
    *   Existence duration: 3 frames.
*   **When `Update()` is called:**
    *   Needs to disappear 3 frames after being created.
*   **When clicked:**
    *   Does nothing.
*   **When colliding with a zombie:**
    *   Within its 3 frames of existence, it will kill all zombies it collides with.

### Repeater Seed

*   **When created:**
    *   Image ID: `IMGID_SEED_REPEATER`.
    *   Position: 5th seed packet location, horizontal spacing 60.
    *   Layer: `LAYER_UI`.
    *   Width: 50, Height: 70.
    *   Animation: None.
    *   Price: 200 sun.
    *   Cooldown time: 240 ticks (8 seconds).
*   **When `Update()` is called:**
    *   The Repeater Seed's `Update()` doesn't necessarily have to do anything...
*   **When clicked:**
    *   If the player is holding a shovel or an unplanted seed, the click is invalid.
    *   If the Repeater Seed is on cooldown, the click is invalid.
    *   Ask `GameWorld` if there are 200 sun points. If yes, spend 200 sun, enter cooldown, generate a Cooldown Mask, and plant a Repeater on the next click on a Planting Spot.

### Repeater

*   **When created:**
    *   Image ID: `IMGID_REPEATER`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_PLANTS`.
    *   Width: 60, Height: 80.
    *   Animation: `ANIMID_IDLE_ANIM`.
    *   HP: 300.
    *   Fires two peas every 30 ticks (1 second).
*   **When `Update()` is called:**
    *   Must first check if it is dead. If dead, wait for `GameWorld` cleanup. `Update()` should return immediately.
    *   If its attack ability is on cooldown, decrease cooldown by 1, `Update()` returns.
    *   If it can attack, it will ask `GameWorld` if there is a zombie to its right in its row. If yes, generate a Pea 30 pixels to its right and 20 pixels above its own coordinates, enter a 30-tick cooldown, and generate another Pea at the same position on the 5th tick after that.
        *   In other words, after firing the second pea, the Repeater's attack ability still needs to cool down for 25 ticks.
*   **When clicked:**
    *   If the player is holding a shovel, the Repeater needs to die. Put down the shovel.

### Shovel

*   **When created:**
    *   Image ID: `IMGID_SHOVEL`.
    *   Position: (x = 600, y = `WINDOW_HEIGHT` – 40).
    *   Layer: `LAYER_UI`.
    *   Width: 50, Height: 50.
    *   Animation: None.
*   **When `Update()` is called:**
    *   Does nothing.
*   **When clicked:**
    *   If the player is holding an unplanted seed, the click is invalid.
    *   If the player is already holding the shovel, put down the shovel.
    *   Otherwise, pick up the shovel. (Implied: player is now "holding" the shovel).

### Regular Zombie

*   **When created:**
    *   Image ID: `IMGID_REGULAR_ZOMBIE`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_ZOMBIES`.
    *   Width: 20, Height: 80. This setting is to prevent it from colliding with two plants simultaneously.
    *   Initial Animation: `ANIMID_WALK_ANIM`.
    *   HP: 200.
*   **When `Update()` is called:**
    *   Must first check if it is dead. If dead, wait for `GameWorld` cleanup. `Update()` should return immediately.
    *   If currently walking, it moves left 1 pixel. If currently eating a plant, it does not move.
*   **When clicked:**
    *   Does nothing.
*   **When colliding:**
    *   If collides with a Pea, takes 20 damage, and the Pea dies.
    *   If collides with an Explosion, dies immediately.
    *   If collides with any plant:
        *   If currently walking, enters eating state and plays `ANIMID_EAT_ANIM` animation.
        *   Deals 3 damage to that plant (this is 3 damage per frame, i.e., 90 damage per second).

### Buckethead Zombie

*   **When created:**
    *   Image ID: `IMGID_BUCKET_HEAD_ZOMBIE`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_ZOMBIES`.
    *   Width: 20, Height: 80.
    *   Initial Animation: `ANIMID_WALK_ANIM`.
    *   HP: 1300 (Bucket 1100 HP, Zombie 200 HP).
*   **When `Update()` is called:**
    *   Must first check if it is dead. If dead, wait for `GameWorld` cleanup. `Update()` should return immediately.
    *   If currently walking, it moves left 1 pixel. If currently eating a plant, it does not move.
    *   If HP is <= 200, it loses the bucket. Change its texture to `IMGID_REGULAR_ZOMBIE`.
*   **When clicked:**
    *   Does nothing.
*   **When colliding:**
    *   If collides with a Pea, takes 20 damage, and the Pea dies.
    *   If collides with an Explosion, dies immediately.
    *   If collides with any plant:
        *   If currently walking, enters eating state and plays `ANIMID_EAT_ANIM` animation.
        *   Deals 3 damage to that plant (3 damage per frame).

### Pole Vaulting Zombie

*   **When created:**
    *   Image ID: `IMGID_POLE_VAULTING_ZOMBIE`.
    *   Position: Determined by the code that creates it.
    *   Layer: `LAYER_ZOMBIES`.
    *   Width: 20, Height: 80.
    *   Initial Animation: `ANIMID_RUN_ANIM`.
    *   HP: 340.
*   **When `Update()` is called:**
    *   Must first check if it is dead. If dead, wait for `GameWorld` cleanup. `Update()` should return immediately.
    *   If currently running, it checks if there is a plant it can jump over 40 pixels to its left:
        a.  In this tick, temporarily move itself 40 pixels to the left.
        b.  Ask `GameWorld` if it is colliding with a plant. If yes, it will:
            *   Stop moving (to align with the animation), play `ANIMID_JUMP_ANIM`. After 42 frames, switch to walking (`ANIMID_WALK_ANIM`) and "teleport" 150 pixels to the left (also to align with animation, as shown in the diagram).
        c.  Regardless of whether it triggered "b" in this frame, don't forget to move back 40 pixels to the right to revert the temporary change in "a".
    *   If currently running, it moves left 2 pixels. If currently walking, it moves left 1 pixel. If currently eating a plant or jumping over a plant, it does not move.
*   **When clicked:**
    *   Does nothing.
*   **When colliding:**
    *   If collides with a Pea, takes 20 damage, and the Pea dies.
    *   If collides with an Explosion, dies immediately.
    *   If collides with any plant (and is not jumping):
        *   If currently walking (after a jump), enters eating state and plays `ANIMID_EAT_ANIM` animation.
        *   Deals 3 damage to that plant (3 damage per frame).
