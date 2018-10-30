# 数据挖掘代码

参考书本,自己修改的代码.

## 第二章节代码

使用数据集,从 1-4 的代码,均采用如下数据集.

```python
users_rating_data = {
    "Angelica": {
        "Blues Traveler": 3.5,
        "Broken Bells": 2.0,
        "Norah Jones": 4.5,
        "Phoenix": 5.0,
        "Slightly Stoopid": 1.5,
        "The Strokes": 2.5,
        "Vampire Weekend": 2.0
    },
    "Bill": {
        "Blues Traveler": 2.0,
        "Broken Bells": 3.5,
        "Deadmau5": 4.0,
        "Phoenix": 2.0,
        "Slightly Stoopid": 3.5,
        "Vampire Weekend": 3.0
    },
    "Chan": {
        "Blues Traveler": 5.0,
        "Broken Bells": 1.0,
        "Deadmau5": 1.0,
        "Norah Jones": 3.0,
        "Phoenix": 5,
        "Slightly Stoopid": 1.0
    },
    "Dan": {
        "Blues Traveler": 3.0,
        "Broken Bells": 4.0,
        "Deadmau5": 4.5,
        "Phoenix": 3.0,
        "Slightly Stoopid": 4.5,
        "The Strokes": 4.0,
        "Vampire Weekend": 2.0
    },
    "Hailey": {
        "Broken Bells": 4.0,
        "Deadmau5": 1.0,
        "Norah Jones": 4.0,
        "The Strokes": 4.0,
        "Vampire Weekend": 1.0
    },
    "Jordyn": {
        "Broken Bells": 4.5,
        "Deadmau5": 4.0,
        "Norah Jones": 5.0,
        "Phoenix": 5.0,
        "Slightly Stoopid": 4.5,
        "The Strokes": 4.0,
        "Vampire Weekend": 4.0
    },
    "Sam": {
        "Blues Traveler": 5.0,
        "Broken Bells": 2.0,
        "Norah Jones": 3.0,
        "Phoenix": 5.0,
        "Slightly Stoopid": 4.0,
        "The Strokes": 5.0
    },
    "Veronica": {
        "Blues Traveler": 3.0,
        "Norah Jones": 5.0,
        "Phoenix": 4.0,
        "Slightly Stoopid": 2.5,
        "The Strokes": 3.0
    }
}
```

### 1. 曼哈顿距离

```python
"""
User rating data
"""


def manhattan_dis(rating1, rating2):
    distance = 0
    for key in rating1:
        if key in rating2:
            distance += abs(rating1[key] - rating2[key])
    return distance


def nearest_user(user):
    min_dis = 0
    round1 = True
    target = None

    for u in users_rating_data:
        if u == user:
            continue
        compare = users_rating_data[u]
        dis = manhattan_dis(users_rating_data[user], compare)
        if round1:
            round1 = False
            min_dis = dis
            target = compare
        if min_dis > dis:
            min_dis = dis
            target = compare
    return target


def recommend(user, nearest):
    arr = []
    for key in nearest:
        if key not in user:
            value = nearest[key]
            arr.append((key, value))

    return sorted(arr, key=lambda score: score[1], reverse=True)


user_key = 'Angelica'
nearest_user = nearest_user(user_key)
print(users_rating_data[user_key])
print(nearest_user)

rcm_value = recommend(users_rating_data[user_key], nearest_user)
print(rcm_value)
```

### 2. 闵可夫斯基距离

```python
def manhattan_dis(rating1, rating2, r):
    distance = 0
    is_pow = False
    for key in rating1:
        if key in rating2:
            distance += pow(abs(rating1[key] - rating2[key]), r)
            is_pow = True
    if is_pow:
        return pow(distance, 1 / r)
    return -1


def nearest_user(user, r):
    min_dis = 0
    round1 = True
    target = None

    for u in users_rating_data:
        if u == user:
            continue
        compare = users_rating_data[u]
        dis = manhattan_dis(users_rating_data[user], compare, r)
        if round1:
            round1 = False
            min_dis = dis
            target = compare
        if min_dis > dis:
            min_dis = dis
            target = compare
    return target


def recommend(user, nearest):
    arr = []
    for key in nearest:
        if key not in user:
            value = nearest[key]
            arr.append((key, value))

    return sorted(arr, key=lambda score: score[1], reverse=True)


user_key = 'Hailey'
nearest_user = nearest_user(user_key, 1)
print(users_rating_data[user_key])
print(nearest_user)

rcm_value = recommend(users_rating_data[user_key], nearest_user)
print(rcm_value)
```

### 3. 皮尔森相似度

```python
def pearson(user1, user2):
    sum_x = 0
    sum_y = 0
    sum_xy = 0

    sum_x2 = 0
    sum_y2 = 0

    n = 0
    for key in user1:
        if key in user2:
            sum_x = sum_x + user1[key]
            sum_y = sum_y + user2[key]
            sum_xy = sum_xy + (user1[key] * user2[key])

            sum_x2 = sum_x2 + (user1[key] * user1[key])
            sum_y2 = sum_y2 + (user2[key] * user2[key])

            n += 1

    body = math.sqrt((sum_x2 - (sum_x * sum_x) / n)) * math.sqrt((sum_y2 - (sum_y * sum_y / n)))
    if body == 0:
        return 0
    else:
        head = sum_xy - (sum_x * sum_y) / n
        return head / body


def get_user(key):
    return users_rating_data[key]


sam = get_user("Angelica")
dan = get_user("Hailey")

dis = pearson(sam, dan)

print(dis)
```

### 4. 余弦相似度

```python
def cosine(user1, user2):
    sum_xy = 0
    sum_x2 = 0
    sum_y2 = 0

    for key in user1:
        if key in user2:
            sum_xy += user1[key] * user2[key]
            sum_x2 += user1[key] * user1[key]
            sum_y2 += user2[key] * user2[key]
        else:
            sum_xy += 0
            sum_y2 += 0
            sum_x2 += user1[key] * user1[key]

    body = math.sqrt(sum_x2) * math.sqrt(sum_y2)

    if 0 == body:
        return 0
    else:
        return sum_xy / body


def get_user(key):
    return users_rating_data[key]


sam = get_user("Angelica")
dan = get_user("Veronica")

dis = cosine(sam, dan)

print(dis)
```

---

## K-means

kmeans 使用数据集如下

```js
breed,height (inches),weight (pounds)
Border Collie,20,45
Boston Terrier,16,20
Brittany Spaniel,18,35
Bullmastiff,27,120
Chihuahua,8,8
German Shepherd,25,78
Golden Retriever,23,70
Great Dane,32,160
Portuguese Water Dog,21,50
Standard Poodle,19,65
Yorkshire Terrier,6,7
```

### 1. kmeans

```py
import math
import random

"""
Implementation of the K-means algorithm
for the book A Programmer's Guide to Data Mining"
http://www.guidetodatamining.com

"""


def getMedian(alist):
    print(alist)
    """get median of list"""
    tmp = list(alist)
    tmp.sort()
    alen = len(tmp)
    if (alen % 2) == 1:
        return tmp[alen // 2]
    else:
        return (tmp[alen // 2] + tmp[(alen // 2) - 1]) / 2


def normalizeColumn(column):
    """normalize the values of a column using Modified Standard Score
    that is (each value - median) / (absolute standard deviation)"""
    median = getMedian(column)
    asd = sum([abs(x - median) for x in column]) / len(column)
    result = [(x - median) / asd for x in column]
    return result


class kClusterer:
    """ Implementation of kMeans Clustering
    This clusterer assumes that the first column of the data is a label
    not used in the clustering. The other columns contain numeric data
    """

    def __init__(self, filename, k):
        """ k is the number of clusters to make
        This init method:
           1. reads the data from the file named filename
           2. stores that data by column in self.data
           3. normalizes the data using Modified Standard Score
           4. randomly selects the initial centroids
           5. assigns points to clusters associated with those centroids
        """
        file = open(filename)
        self.data = {}
        self.k = k
        self.counter = 0
        self.iterationNumber = 0
        # used to keep track of % of points that change cluster membership
        # in an iteration
        self.pointsChanged = 0
        # Sum of Squared Error
        self.sse = 0
        #
        # read data from file
        #
        lines = file.readlines()
        file.close()
        header = lines[0].split(',')
        self.cols = len(header)
        self.data = [[] for i in range(len(header))]
        # we are storing the data by column.
        # For example, self.data[0] is the data from column 0.
        # self.data[0][10] is the column 0 value of item 10.
        for line in lines[1:]:
            cells = line.split(',')
            toggle = 0
            for cell in range(self.cols):
                if toggle == 0:
                    self.data[cell].append(cells[cell])
                    toggle = 1
                else:
                    self.data[cell].append(float(cells[cell]))

        self.datasize = len(self.data[1])
        self.memberOf = [-1 for x in range(len(self.data[1]))]
        #
        # now normalize number columns
        #
        for i in range(1, self.cols):
            self.data[i] = normalizeColumn(self.data[i])

        # select random centroids from existing points
        random.seed()
        self.centroids = [[self.data[i][r] for i in range(1, len(self.data))]
                          for r in random.sample(range(len(self.data[0])),
                                                 self.k)]
        self.assignPointsToCluster()

    def updateCentroids(self):
        """Using the points in the clusters, determine the centroid
        (mean point) of each cluster"""
        members = [self.memberOf.count(i) for i in range(len(self.centroids))]
        self.centroids = [[sum([self.data[k][i]
                                for i in range(len(self.data[0]))
                                if self.memberOf[i] == centroid]) / members[centroid]
                           for k in range(1, len(self.data))]
                          for centroid in range(len(self.centroids))]

    def assignPointToCluster(self, i):
        """ assign point to cluster based on distance from centroids"""
        min = 999999
        clusterNum = -1
        for centroid in range(self.k):
            dist = self.euclideanDistance(i, centroid)
            if dist < min:
                min = dist
                clusterNum = centroid
        # here is where I will keep track of changing points
        if clusterNum != self.memberOf[i]:
            self.pointsChanged += 1
        # add square of distance to running sum of squared error
        self.sse += min ** 2
        return clusterNum

    def assignPointsToCluster(self):
        """ assign each data point to a cluster"""
        self.pointsChanged = 0
        self.sse = 0
        self.memberOf = [self.assignPointToCluster(i)
                         for i in range(len(self.data[1]))]

    def euclideanDistance(self, i, j):
        """ compute distance of point i from centroid j"""
        sumSquares = 0
        for k in range(1, self.cols):
            sumSquares += (self.data[k][i] - self.centroids[j][k - 1]) ** 2
        return math.sqrt(sumSquares)
        # return sumSquares

    def kCluster(self):
        """the method that actually performs the clustering
        As you can see this method repeatedly
            updates the centroids by computing the mean point of each cluster
            re-assign the points to clusters based on these new centroids
        until the number of points that change cluster membership is less than 1%.
        """
        done = False

        while not done:
            self.iterationNumber += 1
            self.updateCentroids()
            self.assignPointsToCluster()
            #
            # we are done if fewer than 1% of the points change clusters
            #
            if float(self.pointsChanged) / len(self.memberOf) < 0.01:
                done = True
        print("Final SSE: %f" % self.sse)

    def showMembers(self):
        """Display the results"""
        for centroid in range(len(self.centroids)):
            print("\n\nClass %i\n========" % centroid)
            for name in [self.data[0][i] for i in range(len(self.data[0]))
                         if self.memberOf[i] == centroid]:
                print(name)


##
## RUN THE K-MEANS CLUSTERER ON THE DOG DATA USING K = 3
###
# change the path in the following to match where dogs.csv is on your machine
km = kClusterer('d://dogs.csv', 3)
km.kCluster()
km.showMembers()
```

测试结果

```py
Final SSE: 5.243159


Class 0
========
Border Collie
Brittany Spaniel
German Shepherd
Golden Retriever
Portuguese Water Dog
Standard Poodle


Class 1
========
Bullmastiff
Great Dane


Class 2
========
Boston Terrier
Chihuahua
Yorkshire Terrier
```

### 2. kmeans++

```py
import math
import random

"""
Implementation of the K-means++ algorithm
for the book A Programmer's Guide to Data Mining"
http://www.guidetodatamining.com

"""


def getMedian(alist):
    """get median of list"""
    tmp = list(alist)
    tmp.sort()
    alen = len(tmp)
    if (alen % 2) == 1:
        return tmp[alen // 2]
    else:
        return (tmp[alen // 2] + tmp[(alen // 2) - 1]) / 2


def normalizeColumn(column):
    """normalize the values of a column using Modified Standard Score
    that is (each value - median) / (absolute standard deviation)"""
    median = getMedian(column)
    asd = sum([abs(x - median) for x in column]) / len(column)
    result = [(x - median) / asd for x in column]
    return result


class kClusterer:
    """ Implementation of kMeans Clustering
    This clusterer assumes that the first column of the data is a label
    not used in the clustering. The other columns contain numeric data
    """

    def __init__(self, filename, k):
        """ k is the number of clusters to make
        This init method:
           1. reads the data from the file named filename
           2. stores that data by column in self.data
           3. normalizes the data using Modified Standard Score
           4. randomly selects the initial centroids
           5. assigns points to clusters associated with those centroids
        """
        file = open(filename)
        self.data = {}
        self.k = k
        self.counter = 0
        self.iterationNumber = 0
        # used to keep track of % of points that change cluster membership
        # in an iteration
        self.pointsChanged = 0
        # Sum of Squared Error
        self.sse = 0
        #
        # read data from file
        #
        lines = file.readlines()
        file.close()
        header = lines[0].split(',')
        self.cols = len(header)
        self.data = [[] for i in range(len(header))]
        # we are storing the data by column.
        # For example, self.data[0] is the data from column 0.
        # self.data[0][10] is the column 0 value of item 10.
        for line in lines[1:]:
            cells = line.split(',')
            toggle = 0
            for cell in range(self.cols):
                if toggle == 0:
                    self.data[cell].append(cells[cell])
                    toggle = 1
                else:
                    self.data[cell].append(float(cells[cell]))

        self.datasize = len(self.data[1])
        self.memberOf = [-1 for x in range(len(self.data[1]))]
        #
        # now normalize number columns
        #
        for i in range(1, self.cols):
            self.data[i] = normalizeColumn(self.data[i])

        # select random centroids from existing points
        random.seed()
        self.selectInitialCentroids()
        self.assignPointsToCluster()

    def showData(self):
        for i in range(len(self.data[0])):
            print("%20s   %8.4f  %8.4f" %
                  (self.data[0][i], self.data[1][i], self.data[2][i]))

    def distanceToClosestCentroid(self, point, centroidList):
        result = self.eDistance(point, centroidList[0])
        for centroid in centroidList[1:]:
            distance = self.eDistance(point, centroid)
            if distance < result:
                result = distance
        return result

    def selectInitialCentroids(self):
        """implement the k-means++ method of selecting
        the set of initial centroids"""
        centroids = []
        total = 0
        # first step is to select a random first centroid
        current = random.choice(range(len(self.data[0])))
        centroids.append(current)
        # loop to select the rest of the centroids, one at a time
        for i in range(0, self.k - 1):
            # for every point in the data find its distance to
            # the closest centroid
            weights = [self.distanceToClosestCentroid(x, centroids)
                       for x in range(len(self.data[0]))]
            total = sum(weights)
            # instead of raw distances, convert so sum of weight = 1
            weights = [x / total for x in weights]
            #
            # now roll virtual die
            num = random.random()
            total = 0
            x = -1
            # the roulette wheel simulation
            while total < num:
                x += 1
                total += weights[x]
            centroids.append(x)
        self.centroids = [[self.data[i][r] for i in range(1, len(self.data))]
                          for r in centroids]

    def updateCentroids(self):
        """Using the points in the clusters, determine the centroid
        (mean point) of each cluster"""
        members = [self.memberOf.count(i) for i in range(len(self.centroids))]

        self.centroids = [[sum([self.data[k][i]
                                for i in range(len(self.data[0]))
                                if self.memberOf[i] == centroid]) / members[centroid]
                           for k in range(1, len(self.data))]
                          for centroid in range(len(self.centroids))]

    def assignPointToCluster(self, i):
        """ assign point to cluster based on distance from centroids"""
        min = 999999
        clusterNum = -1
        for centroid in range(self.k):
            dist = self.euclideanDistance(i, centroid)
            if dist < min:
                min = dist
                clusterNum = centroid
        # here is where I will keep track of changing points
        if clusterNum != self.memberOf[i]:
            self.pointsChanged += 1
        # add square of distance to running sum of squared error
        self.sse += min ** 2
        return clusterNum

    def assignPointsToCluster(self):
        """ assign each data point to a cluster"""
        self.pointsChanged = 0
        self.sse = 0
        self.memberOf = [self.assignPointToCluster(i)
                         for i in range(len(self.data[1]))]

    def eDistance(self, i, j):
        """ compute distance of point i from centroid j"""
        sumSquares = 0
        for k in range(1, self.cols):
            sumSquares += (self.data[k][i] - self.data[k][j]) ** 2
        return math.sqrt(sumSquares)

    def euclideanDistance(self, i, j):
        """ compute distance of point i from centroid j"""
        sumSquares = 0
        for k in range(1, self.cols):
            sumSquares += (self.data[k][i] - self.centroids[j][k - 1]) ** 2
        return math.sqrt(sumSquares)

    def kCluster(self):
        """the method that actually performs the clustering
        As you can see this method repeatedly
            updates the centroids by computing the mean point of each cluster
            re-assign the points to clusters based on these new centroids
        until the number of points that change cluster membership is less than 1%.
        """
        done = False

        while not done:
            self.iterationNumber += 1
            self.updateCentroids()
            self.assignPointsToCluster()
            #
            # we are done if fewer than 1% of the points change clusters
            #
            if float(self.pointsChanged) / len(self.memberOf) < 0.01:
                done = True
        print("Final SSE: %f" % self.sse)

    def showMembers(self):
        """Display the results"""
        for centroid in range(len(self.centroids)):
            print("\n\nClass %i\n========" % centroid)
            for name in [self.data[0][i] for i in range(len(self.data[0]))
                         if self.memberOf[i] == centroid]:
                print(name)


##
## RUN THE K-MEANS CLUSTERER ON THE DOG DATA USING K = 3
###
km = kClusterer('d://dogs.csv', 3)
km.kCluster()
km.showMembers()
```

测试结果

```py
Final SSE: 5.243159


Class 0
========
Border Collie
Brittany Spaniel
German Shepherd
Golden Retriever
Portuguese Water Dog
Standard Poodle


Class 1
========
Boston Terrier
Chihuahua
Yorkshire Terrier


Class 2
========
Bullmastiff
Great Dane
```
