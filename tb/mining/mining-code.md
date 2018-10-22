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
