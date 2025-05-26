<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>곱셈 게임</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      padding-top: 50px;
      transition: background-color 0.5s;
    }
    #scoreIncrease {
      font-size: 24px;
      color: green;
      height: 30px;
      margin-bottom: 10px;
      opacity: 0;
      position: relative;
      transition: opacity 0.5s ease;
      pointer-events: none; /* 클릭 방지 */
    }
    #scoreIncrease.show {
      opacity: 1;
      animation: floatUpFade 2s forwards;
    }
    @keyframes floatUpFade {
      0% {
        opacity: 1;
        top: 0;
      }
      100% {
        opacity: 0;
        top: -30px;
      }
    }
    input {
      font-size: 20px;
      padding: 8px;
      margin: 5px;
    }
    .question, .correctnumber, #second {
      font-size: 28px;
      margin: 15px 0;
    }
    .controls {
      margin-bottom: 20px;
    }
    .hidden {
      display: none;
    }
    button {
      font-size: 18px;
      padding: 10px 20px;
      cursor: pointer;
    }
    .summary {
      font-size: 20px;
      margin-top: 30px;
    }
    #scoreDisplay {
    font-weight: bold;
    font-size: 24px;
    color: #333;
    margin-bottom: 15px;
  } 
  </style>
</head>
<body>
  <div class="controls" id="controls">
    <label>최소값: <input type="number" id="minValue" value="1"></label>
    <label>최대값: <input type="number" id="maxValue" value="25"></label><br>
    <label>목숨 개수: <input type="number" id="lifeCount" value="3"></label>
    <label>문제당 시간 제한(초): <input type="number" id="timeLimit" value="5"></label>
    <label>무제한 목숨 <input type="checkbox" id="unlimitedLives"></label><br>
    <label>무제한 시간 <input type="checkbox" id="unlimitedTime"></label>
    <label>제곱 문제만: <input type="checkbox" id="samenumbercheckbox"></label><br><br>
    <button onclick="startGame()">게임 시작</button>
  </div>

  <div class="question hidden" id="question">문제</div>
  <div class="correctnumber hidden" id="correctnumber"></div>
  <div id="second" class="hidden"></div>
  <div id="lifeDisplay" class="hidden"></div>
  <div id="scoreDisplay" class="hidden"></div>
  <input type="number" id="answer" placeholder="정답 입력" class="hidden" />
  <button id="dontKnowButton" class="hidden" onclick="handleWrongAnswer()">모름</button>
  <div id="scoreIncrease" style="font-size:20px; color:green; height: 30px; margin-bottom: 10px;"></div>
  <div id="summary" class="summary hidden"></div>

  <script>
    let num1, num2;
    let correctCount = 0, totalCount = 0, totalTimeSpent = 0;
    let min, max;
    let lives = 3, originalLives = 3;
    let startTime, timer, timeout;
    let score = 0;
    let answered = false;
    let comboCount = 0;

    const samenumbercheckbox = document.getElementById('samenumbercheckbox');
    const unlimitedLivesCheckbox = document.getElementById('unlimitedLives');
    const unlimitedTimeCheckbox = document.getElementById('unlimitedTime');

    function calculateScore(n1, n2, timeTaken, comboCount) {
      let baseScore = 500;
      const product = n1 * n2;
      const unitDigit = product % 10;

      // 📉 쉬운 조건 (점수 감소)
      if (n1 > 1 && n1 <= 9 && n2 > 1 && n2 <= 9) baseScore -= 500;
      if (n1 % 10 === 0 || n2 % 10 === 0) baseScore -= 600;
      if (n1 === n2) baseScore -= 100;
      if (n1 === 1 || n2 === 1) {
        baseScore -= 600;
      } else if (n1 === 2 || n2 === 2) {
        baseScore -= 300;
      } else if (n1 === 5 || n2 === 5) {
        baseScore -= 200;
      } else if ((n1 >= 3 && n1 <= 9) || (n2 >= 3 && n2 <= 9)) {
        baseScore -= 200;
      }

      if ((n1 >= 1 && n1 <= 9) || (n2 >= 1 && n2 <= 9)) { 
        baseScore -= 100; // 범위 자체 감점 (중복 감점 조절 가능)
      }


      // 📈 어려운 조건 (점수 증가)
      if (n1 >= 20 || n2 >= 20) baseScore += 300;
      if (n1 >= 20 && n2 >= 20) baseScore += 600;
      if (product >= 100) baseScore += 100;

      // 💨 시간 기반 보너스
      if (timeTaken <= 3) {
        baseScore += 100;
      } else if (timeTaken <= 5) {
        baseScore += 50;
      } else if (timeTaken <= 10) {
        baseScore += 20;
      }

      // 🔢 숫자 특성 보너스
      if (unitDigit >= 6) baseScore += 200;
      if (isPrime(n1) || isPrime(n2)) baseScore += 100;
      if (n1 % 2 === 1 && n2 % 2 === 1) baseScore += 50;

      const comboBonus = Math.min(comboCount * 30, 300);
      baseScore += comboBonus;

      return Math.max(baseScore, 10);
    }

    // 소수 판별 함수
    function isPrime(n) {
      if (n < 2) return false;
      for (let i = 2; i <= Math.sqrt(n); i++) {
        if (n % i === 0) return false;
      }
      return true;
    }



    function getRandomNumber(min, max) {
      return Math.floor(Math.random() * (max - min + 1)) + min;
    }

    function updateScoreDisplay() {
      const scoreDiv = document.getElementById('scoreDisplay');
      scoreDiv.textContent = `점수: ${score}점`;
    }

    function updatetimer() {
      const now = Date.now();
      const elapsed = (now - startTime) / 1000;
      document.getElementById('second').textContent = `${elapsed.toFixed(2)}초`;
    }

    function startGame() {
      min = parseInt(document.getElementById('minValue').value);
      max = parseInt(document.getElementById('maxValue').value);
      lives = parseInt(document.getElementById('lifeCount').value);
      originalLives = lives;
      correctCount = 0;
      totalCount = 0;
      totalTimeSpent = 0;
      score = 0;
      if (isNaN(min) || isNaN(max) || min > max) {
        alert("유효한 최소/최대값을 입력하세요.");
        return;
      }
      document.getElementById('scoreDisplay').classList.remove('hidden');
      document.getElementById('lifeDisplay').classList.remove('hidden');
      document.getElementById('scoreDisplay').classList.remove('hidden');
      document.getElementById('controls').classList.add('hidden');
      document.getElementById('question').classList.remove('hidden');
      document.getElementById('correctnumber').classList.remove('hidden');
      document.getElementById('answer').classList.remove('hidden');
      document.getElementById('dontKnowButton').classList.remove('hidden');
      document.getElementById('second').classList.remove('hidden');
      document.getElementById('lifeDisplay').classList.remove('hidden');
      document.getElementById('summary').classList.add('hidden');

      generateQuestion();
    }

    function generateQuestion() {
      updateScoreDisplay();
      num1 = getRandomNumber(min, max);
      num2 = samenumbercheckbox.checked ? num1 : getRandomNumber(min, max);
      answered = false;

      document.getElementById('question').textContent = `${num1} × ${num2} = ?`;
      document.getElementById('correctnumber').textContent = `맞힌 개수: ${correctCount}`;
      document.getElementById('lifeDisplay').textContent = unlimitedLivesCheckbox.checked ? '목숨: 무제한' : `목숨: ${lives}`;
      document.getElementById('answer').value = '';
      document.getElementById('answer').focus();
      document.getElementById('correctnumber').textContent = '';

      startTime = Date.now();
      clearInterval(timer);
      clearTimeout(timeout);

      if (!unlimitedTimeCheckbox.checked) {
        timer = setInterval(updatetimer, 10);
        const timeLimit = parseInt(document.getElementById('timeLimit').value);
        timeout = setTimeout(() => handleWrongAnswer("시간 초과!"), timeLimit * 1000);
      } else {
        document.getElementById('second').textContent = "시간 제한 없음";
      }
    }

    function checkAnswer(e) {
      if (e.key !== 'Enter' || answered) return;

      const userAnswer = parseInt(document.getElementById('answer').value);
      if (isNaN(userAnswer)) {
        alert("답을 입력해주세요!");
        return;
      }

      const now = Date.now();
      const timeTaken = (now - startTime) / 1000;
      const correctAnswer = num1 * num2;

      if (userAnswer === correctAnswer) {
        answered = true; 
        clearInterval(timer);
        clearTimeout(timeout);
        correctCount++;
        totalTimeSpent += timeTaken;
        totalCount++;
        comboCount++;
        const pointsEarned = calculateScore(num1, num2, timeTaken, comboCount);

        score += pointsEarned;
        updateScoreDisplay();

        const scoreIncreaseDiv = document.getElementById('scoreIncrease');
        scoreIncreaseDiv.textContent = `+${pointsEarned}점!`;
        scoreIncreaseDiv.classList.remove('show'); // 애니메이션 재적용을 위해 클래스 제거
        void scoreIncreaseDiv.offsetWidth; // 리플로우 강제
        scoreIncreaseDiv.classList.add('show');
        // 2초 후 사라지게
        setTimeout(() => {
          scoreIncreaseDiv.classList.remove('show');
          scoreIncreaseDiv.textContent = '';
        }, 2000);

        document.body.style.backgroundColor = 'lightgreen';
        document.getElementById('answer').disabled = true;
        
        setTimeout(() => {
          document.body.style.backgroundColor = 'white';
          document.getElementById('answer').disabled = false;
          generateQuestion();
        }, 500);
      } else {
        comboCount = 0;
        handleWrongAnswer();
        updateScoreDisplay(); 
      }
    }


    function handleWrongAnswer() {
      if (answered) return;
      answered = true;

      clearInterval(timer);
      clearTimeout(timeout);

      const now = Date.now();
      const timeTaken = (now - startTime) / 1000;
      totalTimeSpent += timeTaken;
      totalCount++;

      document.body.style.backgroundColor = 'lightcoral';
      showAnswer();

      if (!unlimitedLivesCheckbox.checked) {
        lives--;
        if (lives <= 0) return endGame();
      }

      document.getElementById('lifeDisplay').textContent = unlimitedLivesCheckbox.checked ? '목숨: 무제한' : `목숨: ${lives}`;
      setTimeout(() => {
        document.body.style.backgroundColor = 'white';
        generateQuestion();
      }, 1000);
    }


    function showAnswer() {
      const correctAnswer = num1 * num2;
      document.getElementById('correctnumber').textContent = `정답은 ${correctAnswer} 입니다.`;
    }

    function endGame() {
      document.body.style.backgroundColor = 'white';
      clearInterval(timer);
      clearTimeout(timeout);

      document.getElementById('question').classList.add('hidden');
      document.getElementById('correctnumber').classList.add('hidden');
      document.getElementById('answer').classList.add('hidden');
      document.getElementById('dontKnowButton').classList.add('hidden');
      document.getElementById('second').classList.add('hidden');
      document.getElementById('lifeDisplay').classList.add('hidden');

      const summaryDiv = document.getElementById('summary');
      const accuracy = totalCount ? ((correctCount / totalCount) * 100).toFixed(1) : 0;
      const averageTime = totalCount ? (totalTimeSpent / totalCount).toFixed(2) : 0;

      summaryDiv.innerHTML = `
        <h2>게임 종료!</h2>
        <p>총 문제 수: ${totalCount}</p>
        <p>맞힌 문제 수: ${correctCount}</p>
        <p>점수: ${score}점</p>
        <button onclick="toggleDetails()">더보기</button>
        <div id="details" class="hidden">
          <p>최소값: ${min}</p>
          <p>최대값: ${max}</p>
          <p>정답률: ${accuracy}%</p>
          <p>평균 시간: ${averageTime}초</p>
          <p>설정된 목숨: ${originalLives}</p>
        </div>
        <br><button onclick="resetGame()">처음으로</button>
      `;
      summaryDiv.classList.remove('hidden');
    }

    function toggleDetails() {
      const details = document.getElementById('details');
      details.classList.toggle('hidden');
    }

    function resetGame() {
      document.getElementById('controls').classList.remove('hidden');
      document.getElementById('summary').classList.add('hidden');
    }

    document.getElementById('answer').addEventListener('keydown', checkAnswer);
  </script>
</body>
</html>
