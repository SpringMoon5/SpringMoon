# SpringMoon
Bounce
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>Jeu d'Esquive</title>
<style>
  body {
    background: #111;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    margin: 0;
  }

  canvas {
    background: #222;
    border: 2px solid white;
  }

  /* Barre de recharge du double saut */
  #rechargeBar {
    position: absolute;
    bottom: 10px;
    left: 10px;
    width: 200px;
    height: 20px;
    background-color: #444;
    border: 1px solid #fff;
  }

  #rechargeBarFilled {
    height: 100%;
    background-color: #0f0; /* Couleur de la barre de recharge */
  }

  /* Affichage du meilleur rang dans le coin bas à droite */
  #bestRank {
    position: absolute;
    bottom: 10px;
    right: 10px;
    color: white;
    font-size: 16px;
    font-family: Arial, sans-serif;
  }
</style>
</head>
<body>

<canvas id="game" width="600" height="300"></canvas>
<div id="rechargeBar"><div id="rechargeBarFilled"></div></div>
<div id="bestRank"></div>

<!-- Sélecteur de cube -->
<div id="selector" style="position:absolute;top:20px;left:50%;transform:translateX(-50%);z-index:10;text-align:center;">
  <div style="margin-bottom:10px;font-size:18px;color:#fff;font-weight:bold;">Choisis un cube :</div>
  <button id="redBtn" onclick="selectType('red')" style="background:red;color:#fff;border:none;padding:10px 14px;font-size:16px;margin:5px;border-radius:8px;"><div>🔴 Rouge</div><div style="font-size:12px;margin-top:5px;">Meilleur : Niveau <span id="redLevel">1</span></div></button>
  <button id="blueBtn" onclick="selectType('blue')" style="background:blue;color:#fff;border:none;padding:10px 14px;font-size:16px;margin:5px;border-radius:8px;"><div>🔵 Bleu</div><div style="font-size:12px;margin-top:5px;">Meilleur : Niveau <span id="blueLevel">1</span></div></button>
  <button id="greenBtn" onclick="selectType('green')" style="background:green;color:#fff;border:none;padding:10px 14px;font-size:16px;margin:5px;border-radius:8px;"><div>🟢 Vert</div><div style="font-size:12px;margin-top:5px;">Meilleur : Niveau <span id="greenLevel">1</span></div></button>
  <button id="yellowBtn" onclick="selectType('yellow')" style="background:gold;color:#000;border:none;padding:10px 14px;font-size:16px;margin:5px;border-radius:8px;"><div>🟡 Jaune</div><div style="font-size:12px;margin-top:5px;">Meilleur : Niveau <span id="yellowLevel">1</span></div></button>
</div>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

// ====== JOUEUR ======
let player = {
  x: canvas.width / 2 - 15,
  y: 220,
  width: 30,
  height: 30,
  velocityY: 0,
  jumping: false,
  doubleJumpUsed: false // Flag pour savoir si le double saut a été utilisé
};

let gravity = 0.8;
let obstacles = [];
let gameOver = false;

// ====== SCORE ======
// Scores par cube (individuels)
  let scores = { red: 0, blue: 0, green: 0, yellow: 0 }; // Scores par cube (individuels)
let bestScore = localStorage.getItem("bestScore") || 0;

// Meilleurs scores pour chaque cube
let bestScores = {
  red: localStorage.getItem("bestScore_red") || 0,
  blue: localStorage.getItem("bestScore_blue") || 0,
  green: localStorage.getItem("bestScore_green") || 0,
  yellow: localStorage.getItem("bestScore_yellow") || 0
};

// ====== DIFFICULTÉ ======
let obstacleSpeed = 6;
let spawnRate = 1500; // Spawn des obstacles toutes les 1,5 secondes initialement

// ====== BOUCLIER / INVINCIBILITÉ (BLEU) ======
let shieldActive = false; // Invincibilité en cours
let shieldReady = true; // Le bouclier est prêt à être utilisé
let shieldDuration = 2000; // Durée du bouclier en ms (2 secondes)
let shieldCooldown = 10000; // Cooldown en ms (10 secondes)
let lastShieldTime = 0; // Moment où le bouclier s'est terminé (début du cooldown)
let rechargeProgress = 1; // La barre est pleine au début (valeur entre 0 et 1)

// ====== VOL (VERT) ======
let flyActive = false;
let flyReady = true;
let flyDuration = 2000; // 2s
let flyCooldown = 7000; // 7s
let lastFlyTime = 0;

// ====== TIR (ROUGE) ======
let redInterval = null;
let lastRedShot = 1;
let redCooldown = 5000; // 5s

// ====== MULTIPLICATEUR DE SCORE (JAUNE) ======
let scoreMultiplierActive = false;
let scoreMultiplierReady = true;
let scoreMultiplierDuration = 3000; // 3s
let scoreMultiplierCooldown = 12000; // 12s
let lastScoreMultiplierTime = 0;
let scoreMultiplier = 1; // Multiplicateur du score (1 = normal, 3 = triple gain)

// ====== COURONNES POUR LES NIVEAUX ======
let currentLevels = { red: 1, blue: 1, green: 1, yellow: 1 };
let levelUpFlags = { red: false, blue: false, green: false, yellow: false };

// ====== NIVEAUX PAR CUBE ======
// Chaque palier de 1000 points augmente d'1 niveau, niveaux 1..100
function getLevelFromScore(s) {
  return Math.min(100, Math.floor(s / 1000) + 1);
}

// ====== SAUT (CLIC / TAP) ======
function jump() {
  if (!player.jumping) {
    // Premier saut
    player.velocityY = -15;
    player.jumping = true;
    // Réinitialise les flags liés au bouclier si besoin
  } else {
    // Si le joueur clique en l'air et que le pouvoir est prêt, l'activer
    activateAbility();
  }
}

// Saut uniquement avec clic gauche
canvas.addEventListener("mousedown", function(e) {
  if (e.button === 0) jump();
});
canvas.addEventListener("touchstart", function(e) {
  e.preventDefault();
  jump();
}, { passive: false });

// Activer le pouvoir avec le clic droit (context menu) sur le canvas
canvas.addEventListener('contextmenu', function(e) {
  e.preventDefault();
  activateAbility();
});

// Activer le bouclier d'invincibilité
// Activer le pouvoir en fonction du type sélectionné
function activateAbility() {
  if (!player.type) return;
  if (player.type === 'blue') {
    if (!shieldReady || shieldActive) return;
    shieldActive = true;
    shieldReady = false;
    setTimeout(function() {
      shieldActive = false;
      lastShieldTime = Date.now();
    }, shieldDuration);
  } else if (player.type === 'green') {
    if (!flyReady || flyActive) return;
    flyActive = true;
    flyReady = false;
    // Désactiver la gravité pendant la durée
    setTimeout(function() {
      flyActive = false;
      lastFlyTime = Date.now();
    }, flyDuration);
  } else if (player.type === 'yellow') {
    if (!scoreMultiplierReady || scoreMultiplierActive) return;
    scoreMultiplierActive = true;
    scoreMultiplierReady = false;
    scoreMultiplier = 3; // Triple le gain de score
    setTimeout(function() {
      scoreMultiplierActive = false;
      scoreMultiplier = 1;
      lastScoreMultiplierTime = Date.now();
    }, scoreMultiplierDuration);
  }
  else if (player.type === 'red') {
    // Tir manuel pour le rouge si recharge prête
    const now = Date.now();
    if (now - lastRedShot >= redCooldown) {
      autoShoot();
      lastRedShot = now;
    }
  }
}

// ====== OBSTACLES ======
function createObstacle() {
  // Ne pas créer d'obstacles tant que le joueur n'a pas choisi son cube
  if (!player.type) return false;

  let side = Math.random() < 0.5 ? "left" : "right";
  obstacles.push({
    x: side === "left" ? -30 : canvas.width,
    y: 220,
    width: 30,
    height: 30,
    speed: side === "left" ? obstacleSpeed : -obstacleSpeed
  });
  return true;
}

// Sélectionner le type de joueur
function selectType(t) {
  player.type = t;
  // Verrouiller le choix : désactiver tous les boutons du sélecteur
  const btns = document.querySelectorAll('#selector button');
  btns.forEach(b => {
    b.disabled = true;
    b.style.opacity = '0.5';
    b.style.cursor = 'not-allowed';
  });
  // Nettoyer état précédent
  if (redInterval) { clearInterval(redInterval); redInterval = null; }

  if (t === 'red') {
    // Passage en mode rouge : tir MANUEL. rendre le tir immédiatement prêt
    lastRedShot = Date.now() - redCooldown;
  } else if (t === 'blue') {
    // réinitialiser bouclier
    shieldActive = false; shieldReady = true; lastShieldTime = 0;
  } else if (t === 'green') {
    flyActive = false; flyReady = true; lastFlyTime = 0;
  } else if (t === 'yellow') {
    // réinitialiser multiplicateur de score
    scoreMultiplierActive = false; scoreMultiplierReady = true; lastScoreMultiplierTime = 0; scoreMultiplier = 1;
  }
}

// Mettre à jour l'affichage des meilleurs niveaux
function updateBestLevelsDisplay() {
  // Mettre à jour les niveaux et détecter les level-ups
  ['red', 'blue', 'green', 'yellow'].forEach(color => {
    const newLevel = getLevelFromScore(bestScores[color]);
    const levelElement = document.getElementById(color + 'Level');
    
    if (newLevel > currentLevels[color]) {
      // Level up !
      currentLevels[color] = newLevel;
      levelUpFlags[color] = true;
      levelElement.textContent = newLevel + " 👑";
    } else {
      levelElement.textContent = newLevel;
    }
  });
}

// Tir automatique pour le cube rouge : neutralise le bloc le plus proche devant le joueur
function autoShoot() {
  if (obstacles.length === 0) return;
  // Trouver l'obstacle le plus proche horizontalement
  let bestIndex = -1;
  let bestDist = Infinity;
  for (let i = 0; i < obstacles.length; i++) {
    const obs = obstacles[i];
    const dx = Math.abs((obs.x + obs.width/2) - (player.x + player.width/2));
    const dy = Math.abs((obs.y + obs.height/2) - (player.y + player.height/2));
    if (dx < 200 && dy < 50 && dx < bestDist) {
      bestDist = dx; bestIndex = i;
    }
  }
  if (bestIndex !== -1) {
    // Supprimer l'obstacle touché
    obstacles.splice(bestIndex, 1);
    // Optionnel : flash ou effet (simple cercle)
    // draw shot effect briefly
  }
}

// ====== BOUCLE DE JEU ======
let gameStartTime = Date.now(); // Enregistrer le moment où le jeu commence

let lastObstacleTime = 0;
let obstacleInterval = setInterval(() => {
  let now = Date.now();
  if (now - lastObstacleTime >= spawnRate) {
    const created = createObstacle();
    if (created) lastObstacleTime = now;
  }
}, 100); // Vérifie chaque 100ms pour éviter les regroupements

// Réduit progressivement le spawnRate toutes les 10 secondes
setInterval(() => {
  let elapsedTime = Date.now() - gameStartTime; // Temps écoulé depuis le début du jeu

  if (elapsedTime >= 10000) { // Si plus de 10 secondes se sont écoulées
    // On diminue le spawnRate de 100 ms toutes les 10 secondes
    spawnRate = Math.max(700, spawnRate - 100); // Ne pas descendre en dessous de 700 ms
    obstacleSpeed += 0.1; // Augmenter la vitesse des obstacles

    // Le spawnRate est mis à jour, le nouvel intervalle s'applique automatiquement

    // Réinitialiser le moment de référence pour la prochaine réduction
    gameStartTime = Date.now();
  }
}, 10000); // Vérifie toutes les 10 secondes (10000 ms)

// ====== FONCTION DE RANG ======
function getRank(score) {
  if (score < 2000) return "Débutant";
  if (score < 5000) return "Novice";
  if (score < 10000) return "Intermédiaire";
  if (score < 20000) return "Avancé";
  if (score < 50000) return "Expert";
  if (score >= 50000) return "Maître";
}

// ====== GAME OVER ======
function gameOverScreen() {
  if (scores[player.type] > bestScores[player.type]) {
    bestScores[player.type] = scores[player.type];
    localStorage.setItem("bestScore_" + player.type, bestScores[player.type]);
  }
  
  if (scores[player.type] > bestScore) {
    bestScore = scores[player.type];
    localStorage.setItem("bestScore", bestScore);
  }
  
  let rank = getRank(scores[player.type]);
  
  alert(
    "💀 GAME OVER\n" +
    "Score : " + scores[player.type] + "\n" +
    "Meilleur score cube : " + bestScores[player.type] + "\n" +
    "Meilleur score global : " + bestScore + "\n" +
    "Rang atteint : " + rank + "\n\n" +
    "Meilleurs niveaux par cube :\n" +
    "🔴 Rouge : " + getLevelFromScore(bestScores.red) + "\n" +
    "🔵 Bleu : " + getLevelFromScore(bestScores.blue) + "\n" +
    "🟢 Vert : " + getLevelFromScore(bestScores.green) + "\n" +
    "🟡 Jaune : " + getLevelFromScore(bestScores.yellow)
  );
  location.reload();
}

// ====== BOUCLE DE JEU ======
function update() {
  if (gameOver) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Score (n'augmente que si le joueur a choisi un cube)
  if (player.type) scores[player.type] += 5 * scoreMultiplier;
  const totalScore = scores.red + scores.blue + scores.green + scores.yellow;
  
  // Mettre à jour l'affichage des couronnes en temps réel
  updateBestLevelsDisplay();

  ctx.fillStyle = "white";
  ctx.font = "16px Arial";
  ctx.fillText("Score Total : " + totalScore, 10, 20);
  ctx.fillText("Meilleur : " + bestScore, 10, 40);
  ctx.fillText("Niveaux R/B/V/J : " + getLevelFromScore(scores.red) + " / " + getLevelFromScore(scores.blue) + " / " + getLevelFromScore(scores.green) + " / " + getLevelFromScore(scores.yellow), 10, 60);
  
  // Afficher le score du cube sélectionné
  if (player.type === 'red') {
    ctx.fillStyle = "red";
    ctx.fillText("🔴 Rouge : " + scores.red, 10, 80);
  } else if (player.type === 'blue') {
    ctx.fillStyle = "lightblue";
    ctx.fillText("🔵 Bleu : " + scores.blue, 10, 80);
  } else if (player.type === 'green') {
    ctx.fillStyle = "lightgreen";
    ctx.fillText("🟢 Vert : " + scores.green, 10, 80);
  } else if (player.type === 'yellow') {
    ctx.fillStyle = "gold";
    ctx.fillText("🟡 Jaune : " + scores.yellow, 10, 80);
  }
  
  // Affichage du multiplicateur de score si actif
  if (scoreMultiplierActive) {
    ctx.fillStyle = "gold";
    ctx.font = "bold 20px Arial";
    ctx.fillText("x" + scoreMultiplier, canvas.width / 2 - 10, 50);
  }

  // Gestion du bouclier / recharge
  // Gestion des capacités et de la recharge selon le type sélectionné
  const now = Date.now();
  if (player.type === 'blue') {
    if (!shieldActive && lastShieldTime && now - lastShieldTime > shieldCooldown) shieldReady = true;
    if (shieldActive) {
      rechargeProgress = 0;
    } else if (!shieldReady) {
      rechargeProgress = (now - lastShieldTime) / shieldCooldown;
      if (rechargeProgress >= 1) { rechargeProgress = 1; shieldReady = true; }
    } else rechargeProgress = 1;
  } else if (player.type === 'green') {
    if (!flyActive && lastFlyTime && now - lastFlyTime > flyCooldown) flyReady = true;
    if (flyActive) {
      rechargeProgress = 0;
    } else if (!flyReady) {
      rechargeProgress = (now - lastFlyTime) / flyCooldown;
      if (rechargeProgress >= 1) { rechargeProgress = 1; flyReady = true; }
    } else rechargeProgress = 1;
  } else if (player.type === 'red') {
    // Progression jusqu'au prochain tir automatique
    const elapsed = now - lastRedShot;
    rechargeProgress = Math.min(1, elapsed / redCooldown);
  } else if (player.type === 'yellow') {
    if (!scoreMultiplierActive && lastScoreMultiplierTime && now - lastScoreMultiplierTime > scoreMultiplierCooldown) scoreMultiplierReady = true;
    if (scoreMultiplierActive) {
      rechargeProgress = 0;
    } else if (!scoreMultiplierReady) {
      rechargeProgress = (now - lastScoreMultiplierTime) / scoreMultiplierCooldown;
      if (rechargeProgress >= 1) { rechargeProgress = 1; scoreMultiplierReady = true; }
    } else rechargeProgress = 1;
  } else {
    rechargeProgress = 1;
  }

  // Mise à jour de la barre de recharge : largeur en pourcentage
  document.getElementById('rechargeBarFilled').style.width = (rechargeProgress * 100) + "%";

  // Joueur
  player.y += player.velocityY;
  player.velocityY += gravity;

  if (player.y >= 220) {
    player.y = 220;
    player.velocityY = 0;
    player.jumping = false;
  }
  // Effets de vol si actif (vert)
  if (player.type === 'green' && flyActive) {
    // Soulever légèrement le joueur et neutraliser la gravité temporairement
    player.y = Math.max(50, player.y - 2);
    player.velocityY = 0;
  }

  // Couleur et indication du joueur selon le type et l'état
  if (player.type === 'red') {
    ctx.fillStyle = "red";
  } else if (player.type === 'blue') {
    // Le cube bleu est toujours bleu
    ctx.fillStyle = "blue";
  } else if (player.type === 'green') {
    // Le cube vert reste vert même pendant le vol
    ctx.fillStyle = "green";
  } else if (player.type === 'yellow') {
    // Le cube jaune devient plus éclatant si le multiplicateur est actif
    ctx.fillStyle = scoreMultiplierActive ? "yellow" : "gold";
  } else {
    ctx.fillStyle = "orange"; // par défaut
  }
  ctx.fillRect(player.x, player.y, player.width, player.height);

  // Indicateur prêt selon le type
  if (player.type === 'blue' && shieldReady && !shieldActive) {
    ctx.fillStyle = "yellow"; ctx.font = "12px Arial"; ctx.fillText("⚡ READY", player.x - 10, player.y - 10);
    ctx.strokeStyle = "yellow"; ctx.lineWidth = 2; ctx.strokeRect(player.x - 5, player.y - 5, player.width + 10, player.height + 10);
  } else if (player.type === 'green' && flyReady && !flyActive) {
    ctx.fillStyle = "yellow"; ctx.font = "12px Arial"; ctx.fillText("🕊 READY", player.x - 12, player.y - 10);
    ctx.strokeStyle = "yellow"; ctx.lineWidth = 2; ctx.strokeRect(player.x - 5, player.y - 5, player.width + 10, player.height + 10);
  } else if (player.type === 'red') {
    // Indiquer si le tir est prêt
    if (rechargeProgress >= 1) {
      ctx.fillStyle = "yellow"; ctx.font = "12px Arial"; ctx.fillText("🔫 READY", player.x - 12, player.y - 10);
      ctx.strokeStyle = "yellow"; ctx.lineWidth = 2; ctx.strokeRect(player.x - 5, player.y - 5, player.width + 10, player.height + 10);
    }
  } else if (player.type === 'yellow' && scoreMultiplierReady && !scoreMultiplierActive) {
    ctx.fillStyle = "white"; ctx.font = "12px Arial"; ctx.fillText("⚡x3 READY", player.x - 20, player.y - 10);
    ctx.strokeStyle = "white"; ctx.lineWidth = 2; ctx.strokeRect(player.x - 5, player.y - 5, player.width + 10, player.height + 10);
  }

  // Obstacles
  // Dessiner et gérer collisions (si bouclier actif, aucune perte)
  obstacles.forEach((obs, index) => {
    obs.x += obs.speed;
    // Couleur des obstacles : bleu si bouclier actif (bleu), sinon blanc
    ctx.fillStyle = (player.type === 'blue' && shieldActive) ? "blue" : "white";
    ctx.fillRect(obs.x, obs.y, obs.width, obs.height);

    // Collision
    if (
      player.x < obs.x + obs.width &&
      player.x + player.width > obs.x &&
      player.y < obs.y + obs.height &&
      player.y + player.height > obs.y
    ) {
      if (player.type === 'blue' && shieldActive) {
        // neutralisé
        obstacles.splice(index, 1);
      } else if (player.type === 'red') {
        // collision normale (si red n'a pas de bouclier) -> game over
        gameOver = true; gameOverScreen();
      } else if (player.type === 'green' && flyActive) {
        // si en vol, éviter la collision (on considère que le joueur est au-dessus)
        // neutraliser
        obstacles.splice(index, 1);
      } else {
        gameOver = true; gameOverScreen();
      }
    }

    if (obs.x < -50 || obs.x > canvas.width + 50) obstacles.splice(index, 1);
  });

  requestAnimationFrame(update);
}

// Afficher les meilleurs niveaux au démarrage
updateBestLevelsDisplay();

update();
</script>

</body>
</html>
