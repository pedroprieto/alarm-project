<template>
  <v-app>
    <v-main>
      <div v-if="auth == null">
        <v-card
          class="mx-auto pa-12 pb-8"
          elevation="8"
          max-width="448"
          rounded="lg"
        >
          <v-form @submit.prevent="login()">
            <div class="text-subtitle-1 text-medium-emphasis">Cuenta</div>

            <v-text-field
              density="compact"
              placeholder="Usuario"
              autocomplete="username"
              prepend-inner-icon="mdi-account-outline"
              variant="outlined"
              v-model="user"
              name="username"
            ></v-text-field>

            <div
              class="text-subtitle-1 text-medium-emphasis d-flex align-center justify-space-between"
            >
              Password
            </div>

            <v-text-field
              :append-inner-icon="visible ? 'mdi-eye-off' : 'mdi-eye'"
              :type="visible ? 'text' : 'password'"
              density="compact"
              placeholder="Password"
              prepend-inner-icon="mdi-lock-outline"
              autocomplete="current-password"
              variant="outlined"
              v-model="pass"
              @click:append-inner="visible = !visible"
            ></v-text-field>

            <v-card v-if="errorMessage" class="mb-12" color="red">
              <v-card-text class="text-medium-emphasis text-caption">
                {{ errorMessage }}
              </v-card-text>
            </v-card>

            <v-btn
              type="submit"
              class="mb-8"
              color="blue"
              size="large"
              variant="tonal"
              block
            >
              Log In
            </v-btn>
          </v-form>
        </v-card>
      </div>

      <div v-else-if="ready">
        <v-card
          class="mx-auto pa-12 pb-8"
          elevation="8"
          max-width="448"
          rounded="lg"
        >
          <v-switch
            @change="updateAlarmState()"
            color="primary"
            v-model="alarm_desired"
            label="Cambiar alarma"
          ></v-switch>
          <v-switch
            color="secondary"
            v-model="alarm_reported"
            label="Estado actual alarma"
            disabled
          ></v-switch>
          <!-- <v-btn -->
          <!--   @click="enable_alarm()" -->
          <!--   class="mb-8" -->
          <!--   color="blue" -->
          <!--   size="large" -->
          <!--   variant="tonal" -->
          <!--   block -->
          <!-- > -->
          <!--   Activar alarma -->
          <!-- </v-btn> -->

          <!-- <v-btn -->
          <!--   @click="disable_alarm()" -->
          <!--   class="mb-8" -->
          <!--   color="blue" -->
          <!--   size="large" -->
          <!--   variant="tonal" -->
          <!--   block -->
          <!-- > -->
          <!--   Desactivar alarma -->
          <!-- </v-btn> -->
          <v-btn
            @click="play_sound()"
            class="mb-8"
            color="blue"
            size="large"
            variant="tonal"
            block
          >
            Activar sonido
          </v-btn>

          <v-btn
            @click="stop_sound()"
            class="mb-8"
            color="blue"
            size="large"
            variant="tonal"
            block
          >
            Parar sonido
          </v-btn>
        </v-card>
      </div>
    </v-main>
  </v-app>
</template>

<script setup>
import { onMounted, ref } from "vue";

import {
  CognitoIdentityProviderClient,
  InitiateAuthCommand,
} from "@aws-sdk/client-cognito-identity-provider";

import { CognitoIdentityClient } from "@aws-sdk/client-cognito-identity";

import {
  IoTDataPlaneClient,
  PublishCommand,
  GetThingShadowCommand,
  UpdateThingShadowCommand,
} from "@aws-sdk/client-iot-data-plane";
import { IoTClient /*AttachPolicyCommand*/ } from "@aws-sdk/client-iot";

import { fromCognitoIdentityPool } from "@aws-sdk/credential-providers";

let auth = ref(null);
let user = ref(null);
let pass = ref(null);
let alarm_desired = ref(null);
let alarm_reported = ref(null);
let ready = ref(false);
let errorMessage = ref("");

const region = "eu-west-1";
const cognitoUserPoolId = "eu-west-1_MyN0QjAxZ";
const cognitoUserPoolClientId = "1q79cvin3kj58corv42lkqrn5f";
const cognitoIdentityPoolId = "eu-west-1:b680ccc9-ed1f-43ac-af7b-524d83785fe0";

async function login() {
  try {
    const config = {
      region,
    };
    const client = new CognitoIdentityProviderClient(config);
    const input = {
      // InitiateAuthRequest
      AuthFlow: "USER_PASSWORD_AUTH",
      AuthParameters: {
        USERNAME: user.value,
        PASSWORD: pass.value,
      },
      ClientId: cognitoUserPoolClientId,
    };
    const command = new InitiateAuthCommand(input);
    const response = await client.send(command);
    auth.value = response.AuthenticationResult.IdToken;
    sessionStorage.setItem("auth", JSON.stringify(auth.value));
    await getAlarmState();
    ready.value = true;
    errorMessage.value = "";
  } catch (e) {
    errorMessage.value = "Error en la autenticación";
  }
}

async function getKeys() {
  let ci = new CognitoIdentityClient({
    credentials: fromCognitoIdentityPool({
      clientConfig: { region },
      identityPoolId: cognitoIdentityPoolId,
      logins: {
        "cognito-idp.eu-west-1.amazonaws.com/eu-west-1_MyN0QjAxZ": auth.value,
      },
    }),
  });
  return ci.config.credentials();
}

// async function policyIot(credentials) {
//   const client = new IoTClient({
//     credentials,
//     region,
//   });
//   const input = {
//     policyName: "alarma_casa",
//     target: credentials.identityId,
//   };
//   const command = new AttachPolicyCommand(input);
//   const res = await client.send(command);
// }

async function send_command_iot(topic, data) {
  try {
    let credentials = await getKeys();
    //await policyIot(credentials);

    const clientIot = new IoTDataPlaneClient({
      credentials,
      region,
    });

    const inputIot = {
      topic,
      payload: JSON.stringify(data),
      contentType: "application/json",
      qos: 0,
    };
    const commandIot = new PublishCommand(inputIot);
    const responseIot = await clientIot.send(commandIot);
  } catch (e) {
    auth.value = null;
    sessionStorage.setItem("auth", JSON.stringify(null));
  }
}

// async function enable_alarm() {
//   return send_command_iot("alarma_casa/alarm_enable", { state: "on" });
// }
// async function disable_alarm() {
//   return send_command_iot("alarma_casa/alarm_enable", { state: "off" });
// }
async function play_sound() {
  return send_command_iot("alarma_casa/play_sound", { state: "on" });
}
async function stop_sound() {
  return send_command_iot("alarma_casa/play_sound", { state: "off" });
}

async function getAlarmState() {
  try {
    let credentials = await getKeys();

    const client = new IoTDataPlaneClient({
      credentials,
      region,
    });

    const input = {
      thingName: "alarma_casa",
      shadowName: "alarm_state_shadow",
    };

    const command = new GetThingShadowCommand(input);
    const response = await client.send(command);
    let utf8decoder = new TextDecoder();
    let state = JSON.parse(utf8decoder.decode(response.payload));
    alarm_desired.value = state.state.desired.alarm_enable;
    alarm_reported.value = state.state.reported.alarm_enable;
  } catch (e) {
    auth.value = null;
    sessionStorage.setItem("auth", JSON.stringify(null));
  }
}

onMounted(async () => {
  auth.value = JSON.parse(sessionStorage.getItem("auth"));

  //const data = new URL(window.location.href).hash.split("=");
  // const data = new URL(window.location.href.replace("#", "?")).searchParams.get(
  //   "id_token",
  // );
  if (auth.value != null) {
    await getAlarmState();
    ready.value = true;
  }
});

async function updateAlarmState() {
  try {
    let credentials = await getKeys();

    const client = new IoTDataPlaneClient({
      credentials,
      region,
    });

    let utf8encoder = new TextEncoder();
    let data = {
      state: {
        desired: {
          alarm_enable: alarm_desired.value,
        },
      },
    };

    const input = {
      thingName: "alarma_casa",
      shadowName: "alarm_state_shadow",
      payload: utf8encoder.encode(JSON.stringify(data)),
    };

    const command = new UpdateThingShadowCommand(input);
    const response = await client.send(command);
  } catch (e) {
    auth.value = null;
    sessionStorage.setItem("auth", JSON.stringify(null));
  }
}
</script>
