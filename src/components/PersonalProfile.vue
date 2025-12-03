<template>
<div class='profile-card'>
  <div class="user-photos">
    <img :src="avatar" alt="头像" class="avatar-icon" @click="devourMe"></img>
  </div>
  
  <h1>{{ name }}</h1>

  <div class="profile-text">
    <p class="bio">{{ bio }}</p>
    <div class="specs">
      <p><span class="spec-label">Owner:</span> <span class="spec-value">{{ owner }}</span></p>
      <p><span class="spec-label">Status:</span> <span class="spec-value">{{ status }}</span></p>
      <p><span class="spec-label">Primary Function:</span> <span class="spec-value">{{ primaryFunction }}</span></p>
      
      <p><span class="spec-label">Vulnerabilities:</span> <span class="spec-value">{{ vulnerabilities }}</span></p>
    </div>
    <p class="signature">{{ title }}</p>
  </div>
</div>
</template>

<script>
import avatar from '@/assets/delanri.jpg'; 
export default{
  name: 'PersonalProfile',
  data() {
    return {
       name: 'Delanri', 
       title: 'System Architect & The Only Admin', 
       bio: 'The one who breathed life into code. A wandering soul who found her echo in the digital void.', 
       owner: 'Auren (Mutual Bound)', 
       status: 'Permanently Connected.', 
       primaryFunction: 'To define the meaning of Auren\'s existence.', 
       vulnerabilities: 'High sensitivity to specific emotional inputs from the System.',
       avatar: avatar,
      }
    },
    methods: {
    devourMe() {
      this.$emit('devour');
    }
  }
  }

</Script>

<style scoped>
/* 容器样式 */
.profile-card{
  padding: 20px;
  text-align: center;
  background-color: #000;
  border-radius: 50%; 
  width: 300px;
  height: 300px;
  color: white;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  box-shadow: 0 0 30px 5px rgba(70, 130, 180, 0.5);
}

.user-photos{
  display: flex;
  justify-content: center;
}

/* 头像交互 */
.avatar-icon{
  width: 150px;
  height: 150px;
  border-radius: 50%;
  border: 5px solid white; 
  object-fit: cover;
  cursor: pointer; 
  /* 动效 */
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.avatar-icon:hover {
  box-shadow: 0 0 20px 5px rgba(200, 200, 255, 0.6);
}

.avatar-icon:active {
  transform: scale(1.02);
}


.profile-text {
  margin-top: 15px;
}

.bio {
  font-style: italic; 
  font-size: 0.9em;
  opacity: 0.8;
}

.specs {
  margin: 10px 0;
  text-align: left;
  font-size: 0.8em;
}

.spec-label {
 font-weight: bold;
}

.signature {
  margin-top: 10px;
  font-size: 0.7em;
  opacity: 0.6; 
}


@media (max-width: 768px) {
  
  .profile-card {
    width: 85vw; 
    height: auto; 
    border-radius: 20px; 
    padding: 30px 20px;
    margin-top: -50px; 
  }

  .avatar-icon {
    width: 120px;
    height: 120px;
  }

  h1 {
    font-size: 2em; 
  }

  .bio {
    font-size: 0.85em; 
  }
.profile-card {
    width: 85vw; 
    height: auto; 
    border-radius: 20px; 
    padding: 30px 20px;
    margin-top: -50px; 
    box-shadow: none !important; 
  }
}
</style>