# survival-game
<!DOCTYPE html>
<html>
<head>
  <title>Deep Sea Survival+</title>

  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no"/>
  <meta name="apple-mobile-web-app-capable" content="yes">
  <link rel="apple-touch-icon" href="https://cdn-icons-png.flaticon.com/512/616/616494.png">

  <style>
    html, body {
      margin:0;
      overflow:hidden;
      background:#001f2f;
      position:fixed;
      touch-action:none;
    }
    canvas { display:block; }
  </style>
</head>
<body>

<canvas id="game"></canvas>

<script>
const c = document.getElementById("game");
const x = c.getContext("2d");

function resize(){
  c.width = innerWidth;
  c.height = innerHeight;
}
resize();
addEventListener("resize", resize);

let state="menu", touch=null, multiTouch=false;

let player,enemies,food,particles,score,combo,comboTimer,difficulty,highScore,slowmo=1;

highScore = localStorage.getItem("fish_plus_high")||0;

function reset(){
  player={x:c.width/2,y:c.height/2,size:15,vx:0,vy:0};
  enemies=[]; food=[]; particles=[];
  score=0; combo=1; comboTimer=0;
  difficulty=1;
}

c.addEventListener("touchstart",e=>{
  if(e.touches.length>1){ multiTouch=true; pulse(); }
  touch=e.touches[0];
  if(state!=="playing"){ reset(); state="playing"; }
});

c.addEventListener("touchmove",e=>{
  touch=e.touches[0];
});

c.addEventListener("touchend",()=>{
  touch=null; multiTouch=false;
});

function vibrate(ms){ if(navigator.vibrate) navigator.vibrate(ms); }

function pulse(){
  enemies.forEach(e=>{
    let dx=e.x-player.x, dy=e.y-player.y;
    let d=Math.hypot(dx,dy);
    if(d<150){
      e.x+=dx*2;
      e.y+=dy*2;
    }
  });
  spawnParticles(player.x,player.y,"#00ffff");
  vibrate(30);
}

function spawnEnemy(){
  let side=Math.random()*4|0, xPos,yPos;
  if(side===0){xPos=0;yPos=Math.random()*c.height;}
  if(side===1){xPos=c.width;yPos=Math.random()*c.height;}
  if(side===2){xPos=Math.random()*c.width;yPos=0;}
  if(side===3){xPos=Math.random()*c.width;yPos=c.height;}

  enemies.push({
    x:xPos,y:yPos,
    size:10+Math.random()*20,
    speed:1+Math.random()*difficulty
  });
}

function spawnFood(){
  food.push({x:Math.random()*c.width,y:Math.random()*c.height,size:5});
}

function spawnParticles(x,y,color){
  for(let i=0;i<12;i++){
    particles.push({
      x,y,
      vx:(Math.random()-0.5)*4,
      vy:(Math.random()-0.5)*4,
      life:30,
      color
    });
  }
}

function update(){
  if(state!=="playing") return;

  difficulty+=0.0008;
  comboTimer--;

  if(comboTimer<=0) combo=1;

  if(touch){
    let dx=touch.clientX-player.x;
    let dy=touch.clientY-player.y;
    player.vx+=dx*0.002;
    player.vy+=dy*0.002;
  }

  player.vx*=0.92;
  player.vy*=0.92;

  player.x+=player.vx*slowmo;
  player.y+=player.vy*slowmo;

  player.x=Math.max(player.size,Math.min(c.width-player.size,player.x));
  player.y=Math.max(player.size,Math.min(c.height-player.size,player.y));

  enemies.forEach(e=>{
    let dx=player.x-e.x, dy=player.y-e.y;
    let d=Math.hypot(dx,dy);

    e.x+=dx/d*e.speed*slowmo;
    e.y+=dy/d*e.speed*slowmo;

    // near miss slow-mo
    if(d < player.size + e.size + 20){
      slowmo=0.5;
      setTimeout(()=>slowmo=1,100);
    }

    if(d < player.size + e.size){
      vibrate(100);
      if(score>highScore){
        highScore=Math.floor(score);
        localStorage.setItem("fish_plus_high",highScore);
      }
      state="gameover";
    }
  });

  food = food.filter(f=>{
    let d=Math.hypot(player.x-f.x,player.y-f.y);
    if(d<player.size+f.size){
      combo++;
      comboTimer=60;
      player.size+=0.4;
      score+=10*combo;
      vibrate(10);
      spawnParticles(f.x,f.y,"#66ffcc");
      return false;
    }
    return true;
  });

  score+=0.05*combo;
}

function drawFish(xp,yp,s){
  x.fillStyle="#00ffff";
  x.beginPath();
  x.ellipse(xp,yp,s,s*0.7,0,0,Math.PI*2);
  x.fill();

  x.beginPath();
  x.moveTo(xp-s,yp);
  x.lineTo(xp-s-10,yp-8);
  x.lineTo(xp-s-10,yp+8);
  x.fill();
}

function draw(){
  x.fillStyle="rgba(0,20,40,0.3)";
  x.fillRect(0,0,c.width,c.height);

  drawFish(player.x,player.y,player.size);

  x.fillStyle="#ff4444";
  enemies.forEach(e=>{
    x.beginPath();
    x.arc(e.x,e.y,e.size,0,Math.PI*2);
    x.fill();
  });

  x.fillStyle="#66ffcc";
  food.forEach(f=>{
    x.beginPath();
    x.arc(f.x,f.y,f.size,0,Math.PI*2);
    x.fill();
  });

  particles = particles.filter(p=>{
    p.x+=p.vx;
    p.y+=p.vy-0.5;
    p.life--;
    x.fillStyle=p.color;
    x.fillRect(p.x,p.y,2,2);
    return p.life>0;
  });

  x.fillStyle="white";
  x.font="18px Arial";
  x.fillText("Score: "+Math.floor(score),15,25);
  x.fillText("Best: "+highScore,15,45);
  x.fillText("Combo: x"+combo,15,65);

  if(state==="menu"){
    center("Deep Sea Survival+\nTap to Swim");
  }
  if(state==="gameover"){
    center("Eaten!\nScore: "+Math.floor(score)+"\nTap Again");
  }
}

function center(t){
  x.fillStyle="white";
  x.font="28px Arial";
  x.textAlign="center";
  t.split("\n").forEach((l,i)=>{
    x.fillText(l,c.width/2,c.height/2+i*30);
  });
  x.textAlign="left";
}

function loop(){
  update();
  draw();
  requestAnimationFrame(loop);
}

setInterval(()=>state==="playing"&&spawnEnemy(),650);
setInterval(()=>state==="playing"&&spawnFood(),450);

reset();
loop();
</script>
</body>
</html>