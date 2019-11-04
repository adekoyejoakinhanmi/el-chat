<template>
  <div class="chat-wrapper">
    <div class="chat-header">
      <h4 class="title">Select & Customize Services</h4>
      <div class="price-wrapper">
        <h4>N{{ currentPlan.price || 0 }}</h4>
      </div>
    </div>
  </div>
</template>

<script>
import {
  ConversationalForm,
  EventDispatcher,
  ChatResponseEvents
} from 'conversational-form';
import { plans } from "@/assets/js/plans";
import { robotImage, userImage } from "@/assets/js/misc";
import { isEqual } from "lodash";

export default {
  name: 'PriceCalculator',
  data() {
    return {
      selectedServices: [],
      chatReplies: null,
      addedPaymentQuestion: false,
      steps: {
        currentStep: 0,
        maxSteps: 0
      }
    }
  },
  computed: {
    currentPlan() {
      if (
        !Object.keys(this.services).length
      ) return 0;
      const servicesJson = JSON.parse(JSON.stringify(this.services));
      const filterPlan = plans.filter(item => {
        if (isEqual(item.description, servicesJson)) {
          return item;
        }
      })
      return filterPlan[0];
    },
    services() {
      if (!this.chatReplies) return {};
      const itemsToCollect = {};
      Object.keys(this.chatReplies)
        .filter(e => e !== 'services')
        .forEach(service => {
          let itemPreference = this.chatReplies[service][0].trim();
          if (service === 'laundry-freq' && itemPreference !== "") {
            itemsToCollect['Laundry'] = itemPreference
          }
          if (service === 'cleaning-roomsize' && itemPreference !== "") {
            itemsToCollect['Rooms'] = itemPreference
          }
          if (service === 'cleaning-frequency' && itemPreference !== "") {
            itemsToCollect['Cleaning'] = itemPreference
          }
          if (service === 'meal-freq' && itemPreference !== "") {
            itemsToCollect['Meals'] = itemPreference
          }
        });
      if (itemsToCollect.Rooms && !itemsToCollect.Cleaning)
        delete itemsToCollect.Rooms;
      return itemsToCollect;
    },
    isPenultimateStep() {
      const {
        maxSteps,
        currentStep
      } = this.steps;
      if ((maxSteps - 2) === currentStep) {
        return true;
      }
      return false;
    }
  },
  mounted() {
    this.setupForm();
    const el = document.createElement('script');
    el.src = 'https://js.paystack.co/v1/inline.js';
    this.$el.appendChild(el);
  },
  methods: {
    updateChatReplies() {
      const formDataSerialized = this.cf.getFormData(true);
      this.chatReplies = formDataSerialized;
    },
    updateSteps() {
      const { maxSteps, step } = this.cf.flowManager;
      this.steps = {
        maxSteps,
        currentStep: step
      };
    },
    userClickedChat(e) {
      const currentTags = this.cf.tags;
      const index = currentTags.map(t => t.name).indexOf(e.detail.name);
      const updated = currentTags.slice(0, index + 1);
      this.cf.chatList.clearFrom(index + 1);
      this.cf.flowManager.setTags(updated);
      this.cf.flowManager.startFrom(index + 1, true);
      this.updateChatReplies();
    },
    initPayment(success, error) {
      const vm = this;
      const setupParams = {
        key: process.env.VUE_APP_PAYSTACK_KEY,
        email: vm.chatReplies.user_email,
        amount: vm.currentPlan.price * 100,
        callback: function () {
          success()
        },
        onClose: () => {
          error()
        }
      };
      const handler = window.PaystackPop.setup(setupParams);
      handler.openIframe();
    },
    addPaymentQuestion() {
      const $vm = this;
      const q = [{
        "tag": "input",
        "type": "radio",
        "id": "init-payment",
        "cf-questions": `Almost there. &&We just need to process your payment and Voila!&& Your services come to N${$vm.currentPlan.price}/month. Begin now?`,
        "cf-label": "Yes",
        "value": true,
        "cf-error": "No payment processed. Retry?"
      }];
      this.addedPaymentQuestion = true;
      this.cf.addTags(q);
    },
    setupForm() {
      const formFields = [{
          "tag": "fieldset",
          "type": "Checkboxes",
          "id": "select-automations",
          "required": true,
          "cf-input-placeholder": "Please select your automations",
          "cf-questions": `Hi there. Welcome to Eden Life &&Which of the following services would like to automate on Eden?`,
          "children": [{
              "tag": "input",
              "type": "checkbox",
              "name": "services",
              "cf-label": "Laundry",
              "value": "Laundry"
            },
            {
              "tag": "input",
              "type": "checkbox",
              "name": "services",
              "value": "Cleaning",
              "cf-label": "Cleaning"
            },
            {
              "tag": "input",
              "type": "checkbox",
              "name": "services",
              "value": "Meals",
              "cf-label": "Meals"
            }
          ]
        },
        {
          "tag": "input",
          "type": "email",
          "id": "user_email",
          "name": "user_email",
          "required": true,
          "cf-questions": "Can I have your email please?",
          "cf-input-placeholder": "you@example.com",
          "cf-error": "Please enter a valid email"
        }
      ];
      const $vm = this;
      const eventHandler = new EventDispatcher();
      eventHandler.addEventListener(
        ChatResponseEvents.USER_ANSWER_CLICKED,
        $vm.UserClickedChat,
        false
      );

      const cf = ConversationalForm.startTheConversation({
        options: {
          submitCallback: this.submitCallback,
          flowStepCallback: this.handleStepFlow,
          loadExternalStyleSheet: false,
          preventAutoFocus: true,
          eventDispatcher: eventHandler,
          robotImage,
          userImage
        },
        tags: formFields
      });
      this.cf = cf;
      this.$el.appendChild(this.cf.el);
    },
    submitCallback() {
      this.cf.addRobotChatResponse("Yay! You are done. Your gardener will be in contact with you sson.")
    },
    handleStepFlow(dto, success, e) {
      const currentQuestionId = dto.tag.id;

      if (currentQuestionId === 'select-automations') {
        this.mapAutomations(dto.tag.value);
      }

      if (currentQuestionId === 'laundry-freq') {
        this.addLaundry('laundry-qty');
      }

      if (currentQuestionId === 'laundry-qty' && this.selectedServices.includes('Cleaning')) {
        this.addCleaning();
      }
      if (currentQuestionId === 'select-rooms') {
        this.addCleaning('select-cleaning-freq');
      }
      const canStartMeals =
        (currentQuestionId === 'laundry-qty' && !this.selectedServices.includes('Cleaning')) ||
        currentQuestionId === 'select-cleaning-freq';

      if (canStartMeals && this.selectedServices.includes('Meals')) {
        this.addMeals();
      }
      if (currentQuestionId === 'meals-pref') {
        this.addMeals('meals-freq');
      }
      if (
        this.isPenultimateStep &&
        this.currentPlan.price &&
        !this.addedPaymentQuestion) {
        this.addPaymentQuestion();
      }

      if (currentQuestionId === 'init-payment') {
        return this.initPayment(success, e);
      }
      this.updateChatReplies();
      this.updateSteps();
      return success();
    },
    mapAutomations(automations) {
      const firstItem = automations[0];
      this.selectedServices = automations;
      if (firstItem === 'Laundry') {
        this.addLaundry();
      } else if (firstItem === 'Cleaning') {
        this.addCleaning();
      } else if (firstItem === 'Meals') {
        this.addMeals();
      }
    },
    addLaundry(questionId) {
      const laundry = [{
          "tag": "fieldset",
          "type": "Radio buttons",
          "id": "laundry-freq",
          "name": "laundry-freq",
          "cf-input-placeholder": "",
          "cf-questions": "Awesome &&How often would like us to come in to pick your laundry?",
          "children": [{
              "tag": "input",
              "type": "radio",
              "name": "laundry",
              "cf-label": "Every 2 weeks",
              "value": "bi-weekly",
              "checked": "checked"
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "laundry",
              "cf-label": "Monthly",
              "value": "monthly"
            }
          ]
        },
        {
          "tag": "input",
          "type": "number",
          "id": "laundry-qty",
          "name": "laundry-qty",
          "cf-input-placeholder": "",
          "cf-questions": "Okay &&How many bags of laundry are we picking {laundry-freq}? &&Note, each bag contains about 35 items."
        }
      ];
      const itemToAdd =
        questionId ?
        laundry.find(item => item.id === questionId) :
        laundry[0];
      this.cf.addTags([itemToAdd]);
    },
    addCleaning(questionId) {
      const hasLaundry = this.selectedServices.includes('Laundry');
      const question =
        hasLaundry ?
        `I've taken down your laundry preferences &&How many rooms are in your house?` :
        `Let me take your cleaning preferences. &&How many rooms are in your house?`;

      const cleaning = [{
          "tag": "fieldset",
          "type": "Radio buttons",
          "id": "select-rooms",
          "cf-input-placeholder": "Select rooms number",
          "cf-questions": question,
          "children": [{
              "tag": "input",
              "type": "radio",
              "name": "cleaning-roomsize",
              "cf-label": "1",
              "value": "1"
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "cleaning-roomsize",
              "cf-label": "2",
              "value": "2"
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "cleaning-roomsize",
              "cf-label": "3",
              "value": 3
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "cleaning-roomsize",
              "cf-label": "4",
              "value": 4
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "cleaning-roomsize",
              "cf-label": "5+",
              "value": "5+"
            }
          ]
        },
        {
          "tag": "fieldset",
          "type": "Radio buttons",
          "id": "select-cleaning-freq",
          "required": "required",
          "cf-input-placeholder": "Pick cleaning frequency",
          "cf-questions": "Great! &&How often would like these spaces cleaned?",
          "children": [{
              "tag": "input",
              "type": "radio",
              "name": "cleaning-frequency",
              "cf-label": "Weekly",
              "value": "weekly",
              "checked": "checked"
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "cleaning-frequency",
              "cf-label": "Monthly",
              "value": "monthly"
            }
          ]
        }
      ];
      const itemToAdd =
        questionId ?
        cleaning.find(item => item.id === questionId) :
        cleaning[0];
      this.cf.addTags([itemToAdd]);
    },
    addMeals(questionId) {
      const meals = [{
          "tag": "fieldset",
          "type": "Checkboxes",
          "id": "meals-pref",
          "cf-input-placeholder": "",
          "cf-questions": "Very well &&What are your food preferences? &&Select all that apply",
          "children": [{
              "tag": "input",
              "type": "checkbox",
              "name": "meal-pref",
              "cf-label": "Foods rich in carbs",
              "value": "carbs-rich"
            },
            {
              "tag": "input",
              "type": "checkbox",
              "name": "meal-pref",
              "cf-label": "Veggies and Fruits",
              "value": "veggies-fruits"
            },
            {
              "tag": "input",
              "type": "checkbox",
              "name": "meal-pref",
              "cf-label": "Mix of everything",
              "value": "mix"
            }
          ]
        },
        {
          "tag": "fieldset",
          "type": "Radio buttons",
          "id": "meals-freq",
          "cf-input-placeholder": "",
          "cf-questions": "Nice! &&How often should we bring these delicacies? &&We can deliver breakfast and lunch daily or deliver 3 meals (1-litre of each) and a litre of juice/smooth every week or two.",
          "children": [{
              "tag": "input",
              "type": "radio",
              "name": "meal-freq",
              "cf-label": "Daily",
              "value": "daily"
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "meal-freq",
              "cf-label": "Weekly",
              "value": "weekly"
            },
            {
              "tag": "input",
              "type": "radio",
              "name": "meal-freq",
              "cf-label": "Every 2 weeks",
              "value": "bi-weekly"
            }
          ]
        },
      ];
      const itemToAdd =
        questionId ?
        meals.find(item => item.id === questionId) :
        meals[0];
      this.cf.addTags([itemToAdd]);
    }
  }
};
</script>