<script setup lang="ts">
import { BehaviorSubject, ReplaySubject } from 'rxjs';
import { onUpdated, ref } from 'vue';
import moment from 'moment';
import { settings } from '../admin-settings';
import { addPoints, DisplayData, getPoints, subscribePoints, zeroData } from '../data';

const props = defineProps({ addAmount: { type: Number, default: 0 } });

const displayData: BehaviorSubject<DisplayData> = subscribePoints();

const showColors = ref(true);

let updated = false;
const updatedSubject = new ReplaySubject<void>();

onUpdated(() => {
    if (!updated) {
        updated = true;
        updatedSubject.next();
    }
})

let first = true;

const displayActualData = ref(displayData.value);
if (displayActualData.value !== zeroData) {
    if (updated) {
        setTimeout(() => {
            displayActualData.value.forEach(actualData => actualData.currentPercentage = actualData.relativePercentage);
        });
    }
    else {
        updatedSubject.subscribe(() => {
        setTimeout(() => {
            displayActualData.value.forEach(actualData => actualData.currentPercentage = actualData.relativePercentage);
        });
    });
    }
}

displayData.subscribe(data => {
  displayActualData.value = data;
  if (updated) {
      if (!first) {
        showColors.value = false;
        }
        setTimeout(() => {
            showColors.value = true;
            displayActualData.value.forEach(actualData => actualData.currentPercentage = actualData.relativePercentage);
        });
    }
    else {
    updatedSubject.subscribe(() => {
        setTimeout(() => {
            showColors.value = true;
            displayActualData.value.forEach(actualData => actualData.currentPercentage = actualData.relativePercentage);
        });
    });
    first = false;
    }
});

const pressedColor = ref('red');

const errorMessage = ref('');
const displayErrorMessage = ref(false);

getPoints();

const addPointToColor = async (color: string) => {
  try {
    if (!!props.addAmount) {
        if (!settings.value.amount || isNaN(settings.value.amount) || typeof(settings.value.amount) !== 'number') {
            if (!displayErrorMessage.value) {
                setTimeout(() => displayErrorMessage.value = true);
            }
            errorMessage.value = 'Please enter a valid amount.';
        } else if (!settings.value.date || isNaN(moment(settings.value.date).toDate().getTime())) {
            if (!displayErrorMessage.value) {
                setTimeout(() => displayErrorMessage.value = true);
            }
            errorMessage.value = 'Please enter a valid date.';
        } else if (!settings.value.reason && (errorMessage.value !== 'Please enter a reason. Press again to send anyway.' || pressedColor.value !== color)) {
            if (!displayErrorMessage.value) {
                setTimeout(() => displayErrorMessage.value = true);
            }
            errorMessage.value = 'Please enter a reason. Press again to send anyway.';
        } else {
            if (displayErrorMessage.value) {
                displayErrorMessage.value = false;
                setTimeout(() => errorMessage.value = '');
            }
            await addPoints(color, settings.value.amount || 0, settings.value.date ? moment(settings.value.date).toDate() : undefined, settings.value.owner, settings.value.reason);
            if (!settings.value.keepAmount) {
                settings.value.amount = 1;
            }
            if (!settings.value.keepDate) {
                const currentDate = moment(new Date()).format('YYYY-MM-DDThh:mm');
                settings.value.date = currentDate;
            }
            if (!settings.value.keepOwner) {
                settings.value.owner = '';
            }
            if (!settings.value.keepReason) {
                settings.value.reason = '';
            }
        }
        pressedColor.value = color;
    }
  } catch (e) {
    console.error(e);
  }
};
</script>

<template>
<div v-if="displayActualData" class="content" v-bind:class="{small: !!addAmount}">
    <div v-for="data in displayActualData" :key="data.color" class="house" v-bind:class="{[data.color]: showColors, clickable: !!addAmount}" @click="addPointToColor(data.color)">
        <div class="points-container">
            <span class="warning-message" v-if="errorMessage" v-bind:class="{hide: !displayErrorMessage || pressedColor !== data.color}">
                {{errorMessage}}
            </span>
            <div class="points" style="height: 0%; transition: height 1s ease-in-out;" v-bind:style="{ height: data.currentPercentage + '%' }">
            </div>
        </div>
        <div class="name">
        {{data.colorString}}
        <div class="score">
            <span class="number">
            {{data.points}}
            </span>
            <span class="badge" v-bind:class="[data.badgeClass]">
            {{data.badgeString}}
            </span>
        </div>
        </div>
    </div>
</div>
</template>

<style scoped lang="scss">
.content {
  display: grid;
  grid-template-rows: 1fr 1fr;
  grid-template-columns: 1fr 1fr 1fr;
  width: calc(100% - 1rem);
  height: calc(85% - 1rem);
  padding: 0.5rem;
  margin: 0;
  border: none;

  &.small {
    height: calc(70% - 1rem);
  }
}

@media (max-aspect-ratio: 1/1) {
  .content {
    height: calc(100% - 15vw - 1rem);

    &.small {
        height: calc(100% - 28vw - 1rem);
    }
  }
}

@media (min-aspect-ratio: 7/2) {
  .content {
    grid-template-rows: 1fr;
    grid-template-columns: 1fr 1fr 1fr 1fr 1fr 1fr;
    grid-template-areas: "a b c d e f";
  }
}

@media (max-aspect-ratio: 1/1) {
  .content {
    grid-template-rows: 1fr 1fr 1fr;
    grid-template-columns: 1fr 1fr;
    grid-template-areas: "a b" "c d" "e f";
  }
}

@media (max-aspect-ratio: 4/9) {
  .content {
    grid-template-rows: 1fr 1fr 1fr 1fr 1fr 1fr;
    grid-template-columns: 1fr;
    grid-template-areas: "a" "b" "c" "d" "e" "f";
  }
}

.house {
  position: relative;
  background-color: rgb(238, 238, 238);
  height: auto;
  width: auto;
  display: flex;
  flex-direction: column;
  justify-content: center;
  margin: 0.5rem;
  border: 0.025rem solid rgba(0,0,0,0.5);
  border-radius: 1rem;
  box-shadow: 0 1rem 1rem rgba(0,0,0,0.3);
  transition: transform .2s ease-in-out, background-color .2s ease-in-out;

  &.clickable {
    cursor: pointer;
  }

  &:hover {
    color: #e6e6e6;
    transform: scale(1.03);
  }
}

.red {
  background-color: #AB0200;
}

.blue {
  background-color: #166CC2;
}

.purple {
  background-color: #420082;
}

.grey {
  background-color: #808184;
}

.orange {
  background-color: #FF6228;
}

.yellow {
  background-color: #F6AA00;
}

.points-container {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  padding: 0;
  margin: 0;
  height: 100%;
  border: none;
  border-radius: 1rem;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  overflow: hidden;
  
  .warning-message {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    padding: 0;
    margin: 0;
    width: calc(100% - 2rem);
    text-align: center;
    height: calc(100% - 2rem);
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 1rem;
    background-color: rgba(184, 0, 0, 0.7);
    border-radius: 1rem;
    z-index: 50;
    opacity: 1;
    transition: opacity .3s ease-in-out;

    &.hide {
        opacity: 0;
    }
  }

  .points {
    background-color: #D3D3D3;
    opacity: 0.35;
    padding: 0;
    margin: 0;
    border: none;
  }
}

.name, .score {
  text-align: center;
  padding: 0;
  margin: 0;
  border: none;
}

.name {
  margin-left: 5%;
  margin-right: 5%;
  font-weight: normal;
  font-size: 2rem;
}

.score {
  margin-top: 0.25rem;
  font-weight: bold;
  font-size: 1.5rem;
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
}

.badge {
  background-color: #606060;
  padding: 0.25rem;
  font-size: 0.7rem;
  line-height: 1rem;
  border-radius: 0.6rem;
  margin: 0.1rem;
  margin-left: 0.35rem;
  border: none;
  box-shadow: 0 0.125rem 0.125rem rgba(0,0,0,0.3);
}

.badge.first {
  background-color: #FFD700;
  color: #0f0f0f;
}

.badge.second {
  background-color: #C0C0C0;
  color: #0f0f0f;
}

.badge.third {
  background-color: #cd7f32;
  color: #0f0f0f;
}

.badge.last {
  background-color: #871010;
}
</style>