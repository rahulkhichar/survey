This dynamic survey form system enables clients to create and configure surveys where individual questions can have multiple options, leading to different subsequent questions based on the user's selection, thus supporting complex branching logic. Clients can then circulate these customised surveys across various channels for users to complete. As users fill out the form step-by-step, the system must display a real-time progress indicator, showing "how much of the form is pending to complete" based on the longest possible question path from their current position. Finally, the system must provide clients with comprehensive access to view and analyse all user-filled responses.


Functional Requirements:

For Client:
   1. We should provide functionality to the client so they can create and publish the survey
   2. Client can see analytics of the survey published 
   3. They can see the list of survey forms created by them 
   4. They can see the list of users who filled the survey and their respective responses

For User:
   1. User will start the survey and begin filling the form with visibility to see how much percentage they have to complete the form
   2. At the end, they submit the form

## Implementation Notes

### Progress Calculation Algorithm

**Problem**: We need to show users how much of the survey they still need to complete.

**Solution**: This is a one-time calculation that happens when publishing the survey. Since we know the depth at each question level, we calculate this on the frontend side while publishing the survey and then store it in S3.

Here is the algorithm that helps us compute the completion percentage:

```java
package Algorithm;

import java.util.ArrayList;
import java.util.List;

public class DynamicSurvey {

    public static void main(String[] args) {
        DynamicSurvey dynamicSurvey = new DynamicSurvey();

        Question question1 = new Question(1, "Question First", null);
        Question question2 = new Question(2, "Testing 2", null);
        Question question3 = new Question(3, "Testing 4", null);
        Question question4 = new Question(4, "Description Testing", null);
        Question question5 = new Question(5, "Option Testing", null);

        question1.setOption(new Options(11, "Get the question", question2));
        question1.setOption(new Options(12, "Get the question", question2));
        question1.setOption(new Options(13, "Dummy question", question3));
        question1.setOption(new Options(14, "Done", null));

        question2.setOption(new Options(21, "Get the question", null));
        question2.setOption(new Options(22, "Get the question", question4));
        question2.setOption(new Options(12, "Get the question", question2));

        question3.setOption(new Options(31, "Get the question first", question5));
        question5.setOption(new Options(33, "Choose this one", null));
        
        dynamicSurvey.findTheLengthOfThePaths(question1, 0);
    }

    public int findTheLengthOfThePaths(Question question, int startFromTop) {
        if (question == null) return 0;

        int maximum = 0;

        for (Options options : question.options) {
            maximum = Math.max(findTheLengthOfThePaths(options.getToQuestion(), startFromTop + 1), maximum);
        }

        question.setPerToComplete((double) (maximum * 100) / (maximum + startFromTop));

        return maximum + 1;
    }
}

class Question {
    int id;
    String description;
    List<Options> options;
    double perToComplete;

    public double getPerToComplete() {
        return perToComplete;
    }

    public void setPerToComplete(double perToComplete) {
        this.perToComplete = perToComplete;
    }

    Question(int id, String description, List<Options> options) {
        this.id = id;
        this.description = description;
        this.options = options;
        if (options == null) {
            this.options = new ArrayList<>();
        }
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public List<Options> getOptions() {
        return options;
    }

    public void setOptions(List<Options> options) {
        this.options = options;
    }

    public void setOption(Options options) {
        this.options.add(options);
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }
}

class Options {
    int id;
    String desc;
    Question toQuestion;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getDesc() {
        return desc;
    }

    public void setDesc(String desc) {
        this.desc = desc;
    }

    public Question getToQuestion() {
        return toQuestion;
    }

    public void setToQuestion(Question toQuestion) {
        this.toQuestion = toQuestion;
    }

    Options(int id, String desc, Question toQuestion) {
        this.id = id;
        this.desc = desc;
        this.toQuestion = toQuestion;
    }
}
```





## API Endpoints

### For Client

#### 1. Create Survey
- **Path**: `POST /survey/:clientId`
- **Description**: Create a new survey
- **Response**: `{surveyId, status}`
- **Request Body**:
```json
{
  "surveyName": "Customer Feedback on New Feature",
  "description": "Gathering insights on the usability and value of our latest feature release",
  "status": "DRAFT",
  "startQuestion": "q1",
  "graph": {
    "q1": {
      "questionId": "q1",
      "text": "Overall, how satisfied are you with our new feature?",
      "percentageToComplete": 100,
      "options": [
        {
          "optionId": "opt1_1",
          "text": "Don't want to fill",
          "toQuestion": "q2"
        },
        {
          "optionId": "opt1_2",
          "text": "2 - Dissatisfied",
          "toQuestion": "q3"
        },
        {
          "optionId": "opt1_3",
          "text": "3 - Neutral",
          "toQuestion": null
        },
        {
          "optionId": "opt1_4",
          "text": "4 - Satisfied",
          "toQuestion": null
        }
      ]
    },
    "q2": {
      "questionId": "q2",
      "text": "What specific issues did you encounter?",
      "percentageToComplete": 75,
      "options": [
        {
          "optionId": "opt2_1",
          "text": "Don't want to fill",
          "toQuestion": null
        },
        {
          "optionId": "opt2_2",
          "text": "2 - Dissatisfied",
          "toQuestion": null
        },
        {
          "optionId": "opt2_3",
          "text": "3 - Neutral",
          "toQuestion": null
        },
        {
          "optionId": "opt2_4",
          "text": "4 - Satisfied",
          "toQuestion": null
        }
      ]
    },
    "q3": {
      "questionId": "q3",
      "text": "How likely are you to recommend this feature?",
      "percentageToComplete": 50,
      "options": [
        {
          "optionId": "opt3_1",
          "text": "Don't want to fill",
          "toQuestion": "q4"
        },
        {
          "optionId": "opt3_2",
          "text": "2 - Dissatisfied",
          "toQuestion": "q4"
        },
        {
          "optionId": "opt3_3",
          "text": "3 - Neutral",
          "toQuestion": null
        },
        {
          "optionId": "opt3_4",
          "text": "4 - Satisfied",
          "toQuestion": null
        }
      ]
    },
    "q4": {
      "questionId": "q4",
      "text": "What improvements would you suggest?",
      "percentageToComplete": 25.0,
      "options": [
        {
          "optionId": "opt4_1",
          "text": "Don't want to fill",
          "toQuestion": null
        },
        {
          "optionId": "opt4_2",
          "text": "2 - Dissatisfied",
          "toQuestion": null
        },
        {
          "optionId": "opt4_3",
          "text": "3 - Neutral",
          "toQuestion": null
        },
        {
          "optionId": "opt4_4",
          "text": "4 - Satisfied",
          "toQuestion": null
        }
      ]
    }
  }
}
```

#### 2. Update Survey (Draft State Only)
- **Path**: `PUT /survey/:clientId/:surveyId`
- **Description**: Update an existing survey when it's in draft state
- **Request Body**: Same payload as POST

#### 3. Get Survey Form
- **Path**: `GET /survey/:clientId/:surveyId`
- **Description**: Retrieve the survey form details
- **Response**: Same payload structure as published survey

#### 4. Publish Survey
- **Path**: `POST /survey/:clientId/:surveyId/publish`
- **Description**: Publish the survey and get a short URL for distribution
- **Response**:
```json
{
  "shortUrl": "www.xyz.com/surveyForm",
  "status": "PUBLISHED"
}
```






### For User

#### 1. Submit Survey Response
- **Path**: `POST /:surveyId`
- **Description**: Submit survey responses after completing the form
- **Request Body**:
```json
{
  "email": "user@example.com",
  "submissions": [
    {
      "questionId": "q1",
      "chosenOption": "opt1_2"
    },
    {
      "questionId": "q3", 
      "chosenOption": "opt3_1"
    },
    {
      "questionId": "q4",
      "chosenOption": "opt4_3"
    }
  ]
}
```
- **Response**:
```json
{
  "status": "SUCCESS",
  "message": "Survey response submitted successfully"
}
```

#### User Flow:
1. User clicks on the short URL which opens the survey form
2. The form loads with pre-filled questions from the JSON structure
3. User fills out the form step by step with progress tracking
4. User submits the completed form to the backend



## Database Schema

### Survey Table
```jsonx
{
  "surveyId": "UUID",
  "clientId": "UUID",
  "status": "ENUM('DRAFT', 'PUBLISHED', 'FINISHED')",
  "startDate": "Date",
  "url": "S3 URL where we store the question JSON",
  "endDate": "Date",
  "isActive": "Boolean"
}
```

### UserSurvey Table
```json
{
  "id": "UUID",
  "emailId": "user@example.com",
  "surveyId": "UUID (Reference to Survey)",
  "userResponseUrl": "S3 URL where user responses are stored"
}
```


## Analytics Service Schema

### Questions Table
```json
{
  "questionId": "UUID",
  "surveyId": "UUID",
  "description": "String"
}
```

### Options Table
```json
{
  "optionId": "UUID",
  "questionId": "UUID",
  "description": "String",
  "text": "String"
}
```

### UserQuestionAttempt Table
```json
{
  "id": "UUID",
  "emailId": "String",
  "questionId": "UUID",
  "surveyId": "UUID",
  "chosenOptionId": "UUID"
}
```



## Workflow

### 1. Survey Creation and Publishing
1. Client creates a survey from the frontend and saves it to the backend
2. The backend receives the JSON payload and publishes it to S3
3. The S3 URL is saved in the database
4. An event is published, which is later consumed by the reports service to process the JSON

### 2. Survey Distribution and Response Collection
1. The survey is published, which creates a short URL (internally containing the S3 URL)
2. When users access the short URL, the survey loads in the frontend
  3. Users are prompted to enter their email ID
  4. After submitting the form, the JSON response is sent from the frontend and stored in the UserSurvey table
   - **API Reference**: `POST /:surveyId` (Submit Survey Response endpoint)
  5. This triggers an event that is later consumed by the analytics service
6. The analytics service processes the JSON message and loads the data into tables for analytical purposes



