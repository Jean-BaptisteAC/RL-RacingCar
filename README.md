# :red_car: Reinforcement Learning in a 2D Racing Game

In this project we are going to implement a Deep Reinforcement Learning algorithm in a 2D Racing car game.
The goal of the project is to be able to build and train an autonomous AI that is capable to drive along the track. 

## Using the code

You can use the jupyter version of the project with the file ```AsynchronousDoubleDeepQLearning.ipynb```.
The code is quite long but is splitted in several classes that I will explain in this file.

Other useful documents can be found in the **Resources** Folder.

## Car Environment
Credits for the Environment: 

GitHub: https://github.com/techwithtim/Pygame-Car-Racer

Youtube: https://www.youtube.com/watch?v=L3ktUWfAMPg&list=PLzMcBGfZo4-kmY7Nh4kI9kPPnxJ5JMRPj&index=2

### Car

```
class PlayerCar:
    def __init__(self, max_vel, rotation_vel, track_mask):
        img = pygame.image.load("imgs/purple-car.png")
        factor = 0.25
        size = round(img.get_width() * factor), round(img.get_height() * factor)
        self.img = pygame.transform.scale(img, size)
        self.pixel = pygame.image.load("imgs/pixel.png")
        self.max_vel = max_vel
        self.vel = 0
        self.rotation_vel = rotation_vel
        self.angle = 0
        self.start_pos = (172, 200)
        self.x, self.y = self.start_pos
        size = self.img.get_size()
        radians = math.radians(self.angle)
        self.centerx = self.x + size[0]/2
        self.centery = self.y + size[1]/2
        self.track_mask = track_mask
        self.crash = False
        
        self.acceleration = 0.1
        self.ANGLES = [0, 22.5, -22.5, 45, -45, 90, -90]
        self.distanceMax = 144

    def rotate(self, left=False, right=False):
        if left:
            self.angle += self.rotation_vel
        elif right:
            self.angle -= self.rotation_vel
        if self.angle >= 360:
            self.angle -= 360
        elif self.angle <= -360 :
            self.angle += 360


    def move_forward(self):
        self.vel = min(self.vel + self.acceleration, self.max_vel)
        self.move()

    def move_backward(self):
        self.vel = max(self.vel - self.acceleration, -self.max_vel/2)
        self.move()
        
    def reduce_speed(self):
        self.vel = max(self.vel - self.acceleration / 2, 0)
        self.move()

    def bounce(self):
        self.vel = -self.vel/2
        self.move()

    def move(self):
        radians = math.radians(self.angle)
        vertical = math.cos(radians) * self.vel
        horizontal = math.sin(radians) * self.vel

        self.y -= vertical
        self.x -= horizontal
        
        size = self.img.get_size()
        self.centerx = self.x + size[0]/2
        self.centery = self.y + size[1]/2

    def collide(self, mask, x=0, y=0):
        car_mask = pygame.mask.from_surface(self.img)
        offset = (int(self.x - x), int(self.y - y))
        poi = mask.overlap(car_mask, offset)
        return poi
    
    def handle_collision(self):
        if self.collide(self.track_mask) != None:
            self.bounce()
            self.crash = True

    def reset(self):
        self.x, self.y = self.start_pos
        self.angle = 0
        self.vel = 0
        self.crash = False
        
    def lidarInput(self, x=0, y=0):
        pixel_mask = pygame.mask.from_surface(self.pixel)
        mask = self.track_mask
        
        poi_List = []
        for angle in self.ANGLES:
            radians = math.radians(self.angle + angle)
            vertical = math.cos(radians)
            horizontal = math.sin(radians)
    
            offset = (int(self.centerx - x), int(self.centery - y))
            poi = mask.overlap(pixel_mask, offset)
            
            d = 0
            while poi == None and d <= self.distanceMax:
                y += vertical
                x += horizontal
                offset = (int(self.centerx - x), int(self.centery - y))
                poi = mask.overlap(pixel_mask, offset)
                d += 1
            
            poi_List.append((self.centerx - x, self.centery - y))
            x=0
            y=0
            d=0
            
        return poi_List
    
    
    def lidarMinDistance(self, x=0, y=0):
        pixel_mask = pygame.mask.from_surface(self.pixel)
        mask = self.track_mask
        
        angles = [(-180 + i*22.5) for i in range(0,16)]
        poi_List = []
        for angle in angles:
            radians = math.radians(self.angle + angle)
            vertical = math.cos(radians)
            horizontal = math.sin(radians)
    
            offset = (int(self.centerx - x), int(self.centery - y))
            poi = mask.overlap(pixel_mask, offset)
            
            d = 0
            while poi == None and d <= self.distanceMax:
                y += vertical
                x += horizontal
                offset = (int(self.centerx - x), int(self.centery - y))
                poi = mask.overlap(pixel_mask, offset)
                d += 1
            
            poi_List.append((self.centerx - x, self.centery - y))
            x=0
            y=0
            d=0
            
        return poi_List

    def Inputs(self, cluster_angle):
        # LIDAR
        lidarPoints = self.lidarInput()
        distances = []
        center = (self.centerx, self.centery)
        for i in range(len(lidarPoints)):
            point = lidarPoints[i]
            dist = math.dist(center, point)/self.distanceMax
            normalisedDist = (dist - 0.5)*2
            distances.append(normalisedDist)
            
#         # VELOCITY VECTOR
#         radians = math.radians(self.angle)
#         vertical = math.cos(radians) * self.vel / self.max_vel
#         horizontal = math.sin(radians) * self.vel / self.max_vel
#         velocity = [vertical, horizontal]
        
#         # ROAD VECTOR
#         radians = math.radians(cluster_angle)
#         vertical = math.cos(radians)
#         horizontal = math.sin(radians)
#         road_vector = [vertical, horizontal]
        
#         inputs = distances + velocity + road_vector

        inputs = distances
        return inputs
```

### Track

```
class car_environment():
    def __init__(self):
        
        self.GRASS = self.scale_image(pygame.image.load("imgs/grass.jpg"), 2.5)

        self.TRACK_BORDER = self.scale_image(pygame.image.load("imgs/track-border.png"), 0.9)
        self.TRACK_BORDER_MASK = pygame.mask.from_surface(self.TRACK_BORDER)

        self.WIDTH, self.HEIGHT = self.TRACK_BORDER.get_width(), self.TRACK_BORDER.get_height()
        self.WIN = pygame.display.set_mode((self.WIDTH, self.HEIGHT))

        pygame.font.init()
        self.MAIN_FONT = pygame.font.SysFont("comicsans", 20)

        self.FPS = 60
        
        self.run = True
        self.clock = pygame.time.Clock()
        self.images = [(self.GRASS, (0, 0)),(self.TRACK_BORDER, (0, 0))]
        self.player_car = PlayerCar(4,4, self.TRACK_BORDER_MASK)
        
        self.cluster_list = self.init_Clusters()
        
        cluster_angle = self.checkCluster()
        self.observation_space = len(self.player_car.Inputs(cluster_angle))
#         self.action_space = 6
        self.action_space = 5
    
        self.reward2_power = 2
        self.reward2_coef = 0.5
        
        pygame.display.set_caption("Racing Game!")
        
    def scale_image(self, img, factor):
        size = round(img.get_width() * factor), round(img.get_height() * factor)
        return pygame.transform.scale(img, size)
    
    def init_Clusters(self):
        
        # 0°
        forward_Cluster = [
            0, 
            [120, 130, 110, 330], 
            [350, 540, 120, 250], 
            [680, 130, 120, 180], 
            [670, 420, 130, 370], 
            [340, 310, 120, 110]
        ]

        # 180°
        backward_Cluster = [
            180, 
            [230, 20, 110, 340],
            [10, 20, 110, 440], 
            [540, 420, 130, 250]
        ]

        # 90°
        left_Cluster = [
            90, 
            [340, 20, 460, 110],
            [460, 310, 340, 110],
            [120, 20, 110, 110],
            [230, 360, 110, 100]
        ]

        # -90°
        right_Cluster = [
            -90, 
            [340, 210, 340, 100],
            [350, 420, 190, 120],
            [540, 670, 130, 120],
            [200, 690, 150, 100]
        ]

        # 225°
        right_backward_Cluster = [
            225, 
            [10, 460, 340, 230]
        ]

        cluster_list = [
            forward_Cluster, 
            backward_Cluster, 
            left_Cluster,
            right_Cluster, 
            right_backward_Cluster
                       ]
        
        return cluster_list
    
        
    def checkCluster(self):
        for cluster in self.cluster_list:
            for i in range (1, len(cluster)):
                square = cluster[i]
                conditionA = self.player_car.x >= square[0] and self.player_car.x < square[0] + square[2]
                conditionB = self.player_car.y >= square[1] and self.player_car.y < square[1] + square[3]
                if conditionA and conditionB:
                    return cluster[0]
                
    def move_player_human(self):
        keys = pygame.key.get_pressed()
        moved = False

        if keys[pygame.K_q]:
            self.player_car.rotate(left=True)
        if keys[pygame.K_d]:
            self.player_car.rotate(right=True)
        if keys[pygame.K_z]:
            moved = True
            self.player_car.move_forward()
        if keys[pygame.K_s]:
            moved = True
            self.player_car.move_backward()

        if not moved:
            self.player_car.reduce_speed()

    def move_player_robot(self, action):
        moved = False
        if action == 0:
            self.player_car.rotate(left=True)
        if action == 1:
            self.player_car.rotate(left=True)
            self.player_car.move_forward()
            moved = True
        if action == 2:
            self.player_car.move_forward()
            moved = True
        if action == 3:
            self.player_car.rotate(right=True)
            self.player_car.move_forward()
            moved = True
        if action == 4:
            self.player_car.rotate(right=True)
#         if action == 5:
#             pass
        if not moved:
            self.player_car.reduce_speed()
                
                
    def rewardSpeed(self):
        roadAngle = self.checkCluster()
        radians = math.radians(self.player_car.angle - roadAngle)
        reward = self.player_car.vel*math.cos(radians)/self.player_car.max_vel
        return reward

    def rewardDistance(self):
        lidarPoints = self.player_car.lidarMinDistance()
        distances = []
        center = (self.player_car.centerx, self.player_car.centery)
        for i in range(len(lidarPoints)):
            point = lidarPoints[i]
            dist = math.dist(center, point)
            distances.append(dist)
            
        roadRadius = 36
        minimumDistance = min(distances)
        
        reward = abs(1 - (minimumDistance/roadRadius))
        return reward

    def reward(self):
        
        if self.player_car.crash:
            totalReward = -10
            return totalReward
        
        else:
            reward1 = self.rewardSpeed()
            reward2 = self.rewardDistance()
            alpha = 0.5
#             totalReward = reward1 - self.reward2_coef*(reward2**self.reward2_power)
            totalReward = reward1    
            return(totalReward)
        
    
    def reset(self):
        self.player_car.reset()
        cluster_angle = self.checkCluster()
        return self.player_car.Inputs(cluster_angle)

    def step(self, action, human):
        
        self.clock.tick(self.FPS)
        
        self.draw()
        
        if human:
            self.move_player_human()
        else:
            self.move_player_robot(action)
            
        self.player_car.handle_collision()
            
        if self.player_car.crash :
            next_state = self.reset()
            done = True
        else:
            cluster_angle = self.checkCluster()
            next_state = self.player_car.Inputs(cluster_angle)
            done = False
            
        reward = self.reward()
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                run = False
                self.close()
        
        return next_state, reward, done
    
    
    def draw(self): #(win, images, player_car):
        
        # DRAWING OF IMAGES AND LABELS
        # ____________________________
        
        
        for img, pos in self.images:
            self.WIN.blit(img, pos)

        angle_text = self.MAIN_FONT.render(
            f"Angle: {round(self.player_car.angle)}°", 1, (0, 0, 0))
        self.WIN.blit(angle_text, (10, self.HEIGHT - angle_text.get_height() - 90))

        speed_reward_text = self.MAIN_FONT.render(
            f"Speed Reward: {round(self.rewardSpeed(), 1)}", 1, (0, 0, 0))
        self.WIN.blit(speed_reward_text, (10, self.HEIGHT - speed_reward_text.get_height() - 65))

        distance_reward_text = self.MAIN_FONT.render(
            f"Distance Reward: {round(self.rewardDistance(), 1)}", 1, (0, 0, 0))
        self.WIN.blit(distance_reward_text, (10, self.HEIGHT - distance_reward_text.get_height() - 40))

        vel_text = self.MAIN_FONT.render(
            f"Velocity: {round(self.player_car.vel, 1)}px/s", 1, (0, 0, 0))
        self.WIN.blit(vel_text, (10, self.HEIGHT - vel_text.get_height() - 15))
        

        # DRAWING OF CLUSTERS
        # ___________________
        
        size = 3
        self.cluster_list
        
        # Forward
        color = (255,0,0)
        
        for i in range(1, len(self.cluster_list[0])):
            square = self.cluster_list[0][i]
            pygame.draw.rect(self.WIN, color, pygame.Rect(square[0], square[1], square[2], square[3]), size)

        # Backward
        color = (0,255,0)
        for i in range(1, len(self.cluster_list[1])):
            square = self.cluster_list[1][i]
            pygame.draw.rect(self.WIN, color, pygame.Rect(square[0], square[1], square[2], square[3]), size)

        # Left
        color = (255,0,255)
        for i in range(1, len(self.cluster_list[2])):
            square = self.cluster_list[2][i]
            pygame.draw.rect(self.WIN, color, pygame.Rect(square[0], square[1], square[2], square[3]), size)

        # Right
        color = (0,0,255)
        for i in range(1, len(self.cluster_list[3])):
            square = self.cluster_list[3][i]
            pygame.draw.rect(self.WIN, color, pygame.Rect(square[0], square[1], square[2], square[3]), size)

        # Right/Backward
        color = (0,255,255)
        for i in range(1, len(self.cluster_list[4])):
            square = self.cluster_list[4][i]
            pygame.draw.rect(self.WIN, color, pygame.Rect(square[0], square[1], square[2], square[3]), size)

        
        
        # DRAWING OF CAR
        # ___________________
        
        # self.player_car.draw(self.WIN)
        
        rotated_image = pygame.transform.rotate(self.player_car.img, self.player_car.angle)
        new_rect = rotated_image.get_rect(
            center=self.player_car.img.get_rect(topleft=(self.player_car.x, self.player_car.y)).center)
        self.WIN.blit(rotated_image, new_rect.topleft)
        
        color = (255,0,0)
        lidar = self.player_car.lidarInput()
        for point in lidar:
            pygame.draw.line(self.WIN, color, (self.player_car.centerx, self.player_car.centery), (point[0], point[1]), 1)
            
        pygame.display.update()
    
    def close(self):
        self.run = False
        pygame.quit()
```


## Q-Learning Agent
### Agent actions
### Input of the Neural Network
### Rewards

## Multi Agent
### Figure of the system
### Saving and reloading a model

## Results

![image](https://user-images.githubusercontent.com/66775006/215592124-2b5531ca-5a56-460c-821a-2c71b11efd92.png)

![image](https://user-images.githubusercontent.com/66775006/215592143-03864f67-eeb6-4b3d-9f70-5ad04cf728c9.png)

![image](Resources/Animation.gif)

## Conclusion of the project


