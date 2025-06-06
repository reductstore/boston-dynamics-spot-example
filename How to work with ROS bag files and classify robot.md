# How to work with ROS bag files and classify robot movements using Python

Working with real-world robot data often starts with `.bag` files — a standard format for recording ROS (Robot Operating System) messages. Whether you’re logging odometry, camera frames, LiDAR or IMU data, bag files contain valuable data on how a robot interacts with the world.

In this tutorial, we’ll show you how to:

- extract motion data from `.bag` files
- create basic velocity features
- train a classification model to recognize different types of robot movements

We'll use the `bagpy` library to process `.bag` files and basic machine learning techniques for classification.

Although the examples use a Boston Dynamics Spot robot (moving forward, sideways, and rotating), you can adapt the code for your own recordings. The original data can be found [here](http://ptak.felk.cvut.cz/darpa-subt/qualification_videos/spot/).

### **Install Required Libraries**

```python
!pip install numpy pandas matplotlib seaborn scikit-learn bagpy
```

# Loading and Preprocessing Bag Files

Let's create a function to load a `.bag` file and extract velocity features. 

In our example, odometry data is published under the `/spot/odometry` topic. Make sure to specify the correct topic where your robot's motion data is recorded. Depending on your specific use case, you might find other features, such as accelerations or additional sensor data, more relevant for recognizing your robot's movements. For our task, we focus primarily on linear and angular velocities.

```python
import numpy as np
import pandas as pd
from bagpy import bagreader

def process_bag_to_dataframe(bag_path, topic='/spot/odometry', target_label=0):
   
    # Load a ROS bag file and generate a DataFrame with velocity features and a target label.
    
    bag = bagreader(bag_path)
    df = pd.read_csv(bag.message_by_topic(topic))

    # Calculate linear and angular velocities
    df['linear_velocity'] = np.sqrt(df['twist.twist.linear.x']**2 + 
                                    df['twist.twist.linear.y']**2 + 
                                    df['twist.twist.linear.z']**2)

    df['angular_velocity'] = np.sqrt(df['twist.twist.angular.x']**2 + 
                                     df['twist.twist.angular.y']**2 + 
                                     df['twist.twist.angular.z']**2)

    # Assign target label
    df['target'] = target_label

    # Keep only relevant columns
    return df[['twist.twist.linear.x', 'twist.twist.linear.y', 'twist.twist.linear.z',
               'twist.twist.angular.x', 'twist.twist.angular.y', 'twist.twist.angular.z',
               'linear_velocity', 'angular_velocity', 'target']]
```

### **Processing three different movement types**

Let's apply the `process_bag_to_dataframe` function to load and process the data for each of the three movement types. Each movement was recorded in a separate `.bag` file.

```python
df_forward = process_bag_to_dataframe('linear_x.bag', target_label=1)   # moving forward
df_sideways = process_bag_to_dataframe('linear_y.bag', target_label=2)  # moving sideways
df_rotation = process_bag_to_dataframe('rotation.bag', target_label=3)  # rotating

# Combine all samples
df_all = pd.concat([df_forward, df_sideways, df_rotation], ignore_index=True)
```

### Visualizing velocities

We can plot linear and angular velocities over time for each kind of motion, such as the forward movement.

```python
import matplotlib.pyplot as plt

def plot_velocities(df, title=''):

    plt.figure(figsize=(7, 3))

    plt.subplot(1, 2, 1)
    plt.plot(df['linear_velocity'], color='#4B0082')
    plt.title('Linear Velocity')
    plt.xlabel('Time Step')

    plt.subplot(1, 2, 2)
    plt.plot(df['angular_velocity'], color='#9A9E5E')
    plt.title('Angular Velocity')
    plt.xlabel('Time Step')

    plt.suptitle(title)
    plt.tight_layout()
    plt.show()

plot_velocities(df_forward, 'Forward Movement')
```

![Forward Movement](https://github.com/reductstore/boston-dynamics-spot-example/blob/main/forward_movement.png)

# Training and Evaluating Classification Models

We will test several popular models: Logistic Regression, Decision Tree, Random Forest, and Support Vector Machine.

We'll also use `GridSearchCV` for hyperparameter tuning. You can experiment with other hyperparameters to fine-tune the models based on your specific data and requirements.

We evaluate the classifier using **F1 Score**, as it balances precision and recall. This is particularly useful for imbalanced datasets. However, you can also use Accuracy, Precision, and Recall.

Let's prepare the data for training.

```python
X = df_all.drop('target', axis=1)
y = df_all['target']
```

The features `X` consist of the velocity data, while the labels `y` represent the movement types.

Let’s define the scalers, models and hyperparameters.

```python
# Scalers
scalers = {
    'Standard Scaler': StandardScaler(),
    'MinMax Scaler': MinMaxScaler(),
    'Robust Scaler': RobustScaler()
}

# Models
models = {
    'Logistic Regression': LogisticRegression(max_iter=10000),
    'Decision Tree': DecisionTreeClassifier(),
    'Random Forest': RandomForestClassifier(),
    'SVM': SVC(probability=True)
}

# Hyperparameters for tuning
parameters = {
    'Logistic Regression': {'C': [0.01, 0.1, 1, 10, 100]},
    'Decision Tree': {'max_depth': [3, 5, 10, None]},
    'Random Forest': {'n_estimators': [50, 100]},
    'SVM': {'C': [0.1, 1, 10], 'kernel': ['linear', 'rbf']}
}
```

Next, let’s train and test the models.

```python
from sklearn.model_selection import train_test_split, GridSearchCV, StratifiedKFold
from sklearn.preprocessing import StandardScaler, MinMaxScaler, RobustScaler
from sklearn.metrics import f1_score, confusion_matrix
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.ensemble import RandomForestClassifier
from sklearn.svm import SVC

def run_classification(model_name, scaler_name, X, y):

    # Split data into train and test sets
    X_train, X_test, y_train, y_test = train_test_split(X, y, stratify=y, test_size=0.2)
    
    # Apply chosen scaler to the data
    scaler = scalers[scaler_name]
    X_train = scaler.fit_transform(X_train)
    X_test = scaler.transform(X_test)
    
    # Set up hyperparameter grid and cross-validation
    param_grid = parameters[model_name]
    cv = StratifiedKFold(n_splits=5, shuffle=True)
    grid = GridSearchCV(models[model_name], param_grid, cv=cv, scoring='f1_weighted')
    grid.fit(X_train, y_train)
    
    # Get best model from grid search
    best_model = grid.best_estimator_
    
    # Make predictions on test set
    y_pred = best_model.predict(X_test)
    
    print(f'Best parameters: {grid.best_params_}')
    print(f'F1 Score: {f1_score(y_test, y_pred, average="weighted"):.3f}')
    
    return y_test, y_pred, best_model, X.columns
```

We'll plot the confusion matrix to visualize how well our model is performing by comparing the predicted values with the actual labels.

```python
import seaborn as sns

# Plot confusion matrix
def plot_confusion_matrix(y_test, y_pred):

    cm = confusion_matrix(y_test, y_pred)

    sns.heatmap(cm, annot=True, fmt='d', cmap='Purples')
    plt.title('Confusion Matrix')
    plt.xlabel('Predicted')
    plt.ylabel('Actual')
    plt.show()
```

We’ll visualize the feature importance for Decision Tree and Random Forest models to understand which features contribute the most to the model’s predictions.

```python

# Feature importance for Decision Tree and Random Forest
def plot_feature_importance(best_model, X_columns):

    importances = best_model.feature_importances_
    sorted_idx = importances.argsort()[::-1]

    sns.barplot(x=importances[sorted_idx], y=X_columns[sorted_idx], color='#50208B')
    plt.title('Feature Importances')
    plt.xticks(rotation=90)
    plt.tight_layout()
    plt.show()
```

### **Training a Random Forest classifier**

Finally, let's apply the Random Forest classifier to our data and see how well it performs.

```python
y_test, y_pred, best_model, X_columns = run_classification('Random Forest', 'MinMax Scaler', X, y)
```

**Best parameters**: {'n_estimators': 100}

**F1 Score**: 0.976

The confusion matrix provides a breakdown of how the classifier performed on each movement type. The diagonal elements represent the correctly classified instances, while the off-diagonal elements indicate misclassifications. 

```python
plot_confusion_matrix(y_test, y_pred)
```

![Confusion Matrix](https://github.com/reductstore/boston-dynamics-spot-example/blob/main/confusion_matrix.png)

With an F1 score of 0.976, this classifier performs well, but you can always experiment to optimize the model based on your own data.

After evaluating the Random Forest model, we can look at **Feature Importance** to see velocity components contributed most to distinguishing the movement types. This is useful for Decision Tree and Random Forest models, which automatically rank features based on their importance. 

```python
plot_feature_importance(best_model, X_columns)
```

![Feature Importance](https://github.com/reductstore/boston-dynamics-spot-example/blob/main/feature_importance.png)

# Conclusion

In this tutorial, we demonstrated how to process robot movement data from `.bag` files, extract key velocity features, and train machine learning models to classify different movement types. 

Next, you can experiment with different models, hyperparameters, or additional features to further improve classification performance. You can also explore more advanced techniques such as deep learning for more complex tasks.
