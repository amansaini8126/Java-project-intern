import javafx.animation.*;
import javafx.fxml.FXML;
import javafx.scene.control.*;
import java.sql.*;
import java.util.*;

public class QuizController {
    private int userId, quizId, score = 0, currentQuestionIndex = 0, timeLeft;
    private List<Question> questions = new ArrayList<>();
    private Timeline timer;

    @FXML private Label questionLabel, timerLabel;
    @FXML private RadioButton optionA, optionB, optionC, optionD;
    @FXML private Button nextButton;

    public QuizController(int userId, int quizId) {
        this.userId = userId;
        this.quizId = quizId;
    }

    public void startQuiz() {
        loadQuestions();
        Collections.shuffle(questions);  // Randomize
        loadQuizTime();
        startTimer();
        showQuestion();
    }

    private void loadQuestions() {
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement stmt = conn.prepareStatement("SELECT * FROM questions WHERE quiz_id=?")) {
            stmt.setInt(1, quizId);
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                questions.add(new Question(rs.getString("question_text"), rs.getString("option_a"),
                        rs.getString("option_b"), rs.getString("option_c"), rs.getString("option_d"),
                        rs.getString("correct_answer")));
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void loadQuizTime() {
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement stmt = conn.prepareStatement("SELECT time_limit FROM quizzes WHERE id=?")) {
            stmt.setInt(1, quizId);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) timeLeft = rs.getInt("time_limit") * 60;  // Convert to seconds
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void startTimer() {
        timer = new Timeline(new KeyFrame(javafx.util.Duration.seconds(1), e -> {
            timeLeft--;
            timerLabel.setText("Time: " + timeLeft / 60 + ":" + String.format("%02d", timeLeft % 60));
            if (timeLeft <= 0) {
                endQuiz();
            }
        }));
        timer.setCycleCount(Timeline.INDEFINITE);
        timer.play();
    }

    private void showQuestion() {
        if (currentQuestionIndex < questions.size()) {
            Question q = questions.get(currentQuestionIndex);
            questionLabel.setText(q.question);
            optionA.setText(q.optionA);
            optionB.setText(q.optionB);
            optionC.setText(q.optionC);
            optionD.setText(q.optionD);
        } else {
            endQuiz();
        }
    }

    @FXML
    private void handleNext() {
        String selected = getSelectedOption();
        if (selected != null && selected.equals(questions.get(currentQuestionIndex).correctAnswer)) {
            score++;
        }
        currentQuestionIndex++;
        showQuestion();
    }

    private String getSelectedOption() {
        if (optionA.isSelected()) return "A";
        if (optionB.isSelected()) return "B";
        if (optionC.isSelected()) return "C";
        if (optionD.isSelected()) return "D";
        return null;
    }

    private void endQuiz() {
        timer.stop();
        int timeTaken = (questions.size() * 60) - timeLeft;  // Approximate
        storeResult(score, questions.size(), timeTaken);
        displayResults();
    }

    private void storeResult(int score, int total, int timeTaken) {
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement stmt = conn.prepareStatement("INSERT INTO results (user_id, quiz_id, score, total_questions, time_taken) VALUES (?, ?, ?, ?, ?)")) {
            stmt.setInt(1, userId);
            stmt.setInt(2, quizId);
            stmt.setInt(3, score);
            stmt.setInt(4, total);
            stmt.setInt(5, timeTaken);
            stmt.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void displayResults() {
        StringBuilder report = new StringBuilder("Quiz Results:\nScore: " + score + "/" + questions.size() + "\n\nCorrect Answers:\n");
        for (Question q : questions) {
            report.append(q.question).append(" -> ").append(q.correctAnswer).append("\n");
        }
        Alert alert = new Alert(Alert.AlertType.INFORMATION, report.toString());
        alert.show();
    }

    static class Question {
        String question, optionA, optionB, optionC, optionD, correctAnswer;
        Question(String q, String a, String b, String c, String d, String ans) {
            question = q; optionA = a; optionB = b; optionC = c; optionD = d; correctAnswer = ans;
        }
    }
}
