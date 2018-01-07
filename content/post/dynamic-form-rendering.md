---
title: "Dynamic Form Rendering"
date: 2018-01-07T17:38:54+01:00
draft: true
---

A common need in enterprise application development is to support dynamically rendering of a form based on some domain specific metadata. This enables central control of domain logic and enables non-technical domain expert to determine how the application is rendered, commonly using some CMS like GUI presented in the domain language. Furthermore it speeds up lead time of changes because the domain persons can implement changes themselves working with a high level abstractions of the domain.

# The problem

I have worked on implementing such system for support dynamic rendering of a questionnaire, being generated based on metadata provided by domain experts through a central domain service. The architecture of the system was designed according best practises with domains contain micro services, each having a clear central responsibility. The service for providing the quesitonnaire data had contained CRUD functionalities for the questionnaire and was designed with language of it's users. These were strictly domain experts and shouldn't deal with how the questionnaire actually got rendered by the service's clients. Some translation logic was needed to translate the provided questionnaire into a renderable questionnaire. The questionnaire service where responsible for specifying each hos each question:

* Answer type (how should the question be answered)
* Answer options (if any, what answer options should be available)
* Validation rules (how should the question be validated)
* Match criterias (when should the question be displayed)

The model of the question could look like this:
```Javascript

export class Question<T = any> {

    constructor() {
        this.answerOptions = [];
        this.matchCriterias = [];
        this.validationRules = [];
    }

    externalQuestionId: string;
    questionText: string;
    helpText: string;
    answerType: string;
    answerOptions: AnswerOption[];
    matchCriterias: MatchCriteria[];
    validationRules: string[];
    answer?: T;
}
```

So after having discussed the model of the question, how should the form be rendered in the Angular app?

# Reactive forms vs template driven forms

In breif, template driven forms is when the form is defined in the template using the ngForm directive and ngModel for databinding the form data to the bound models. This way form values and validators are set in the template using ngModel and validator directives. With reactive forms a FormGroup is provided using the FormGroup directive containing form state and validation rules, even trough this can require more code and complexity, provides more flexibility as it can by code change form values and validators. Also reactive forms moves logic out of the template to the typescript code, making the application more testable.

Rendering a form dynamically and needing to set validation rules dynamically is recommended to be implemented using reactive forms, as contrast to template driven forms, because:

* Reactive supports adding validation rules dynamically, whereas template driven forms hook up validation rules using directives, which can not be added dynamically (at least if you don’t want to hack around). This is needed when the form is getting rendered and the question's validators needs to be added to the form.
* Reactive forms will support better error handling because it is being mapped and generated in typescript code.
* Reactive forms will move the form and form mapping logic out of the templates and into encapsulated mapping code (or references to this).
* Reactive forms defines the form in typescript code, and not in the html template as template driven forms, which will make the form logic more testable.

# Rendering the questionnaire

Rendering the questionnaire consists of traversing the questionnair's questions and applying the correct mapping of each question.

## Creating the form group
To keep the business logic of the presentation layer the logic for generating the form is extracted in it's own service. The service is responsible generate a form provided an array of questions.\\
This service for each question:\\

* Mapping and setting the validators
* Mapping and setting the answer options as a form group
* Adding the FormGroup (if multianswer quesiton) or FormControl (if single answer question)

```Javascript
    export class QuestionFormGeneratorService {

    constructor() { }
    getQuestionsFormGroup(quesitons: Question[]): FormGroup {

        const group: { [key: string]: AbstractControl } = {};
        quesitons.forEach(question => {
            const validatiors = this.getValidationFunctions(question.validationRules);

            group[question.externalQuestionId] =
                question.answerType.toUpperCase() === selectMulti ?
                    new FormGroup(this.getAnswerOptionsControlsObj(question.answerOptions), validatiors) :
                    new FormControl(question.answer || '', validatiors);
        });
        return new FormGroup(group);
    }

    private getAnswerOptionsControlsObj(answerOptions: AnswerOption[]) {
        const group: { [key: string]: AbstractControl } = {};
        answerOptions.forEach(answerOption => group[answerOption.optionCode]
            = new FormControl());
        return group;
    }

    private getValidationFunctions(validationRuleStrings: string[]): Array<(control: AbstractControl) => ValidationErrors | null> | null {

        const validationRules = [];
        for (const validationRuleStr of validationRuleStrings) {
            const validationRule = ValidationRule.validationRulesMap.get(validationRuleStr.toUpperCase());
            if (validationRule !== undefined) {
                validationRules.push(validationRule.validationFn);
            }
        }

        return validationRules.length > 0 ? validationRules : null;
    }
}

```
[QuestionFormGenerator.service.ts](https://github.com/lydemann/questionnaire-render-demo/blob/master/src/app/questionnaire-section/question-form-generator.service.ts)

### ValidationRules and the object enum pattern

The applications validations rules are in the static map property:
`ValidationRule.validationRulesMap`
ValidationRule implements the "object enum pattern", that is, a pattern for creating enums containing object. The validation-rules class looks like this:
```javascript
export class ValidationRule {

private static _validationRulesMap: Map<string, ValidationRule>;
public static get validationRulesMap() {
    if (!ValidationRule._validationRulesMap) {
        ValidationRule._validationRulesMap = new Map<string, ValidationRule>();
    }

    return ValidationRule._validationRulesMap;
}

public static required = new ValidationRule('REQUIRED', Validators.required);

private constructor(private name: string, public validationFn: (control: AbstractControl) => ValidationErrors | null) {
    ValidationRule.validationRulesMap.set(name, this);
}

toString() {
    return this.name;
}
}
```

Pay attention to the private constructor making sure this class instances is only exposed through the static properties. Also every time a new static property is added it is automatically added to the validationRulesMap in the constructor.
The toString method is used for getting the objects values as a string.

## Mapping to rendering type using the rules pattern

As mentioned before the questions doesn't contain any ui information, only question data such as answer type and answer options. To determine what ui component a question should be rendered as some rules need to be defined for this mapping. The naive approach to this is to clutter a method with all kinds of conditional statements creating bad readability and messy design.\\
The goal here is to ensure satisfaction of the SOLID principles by seperating the actual defining of rules with the actual rules logic, which will support extension of new rules with modifying existing functionality. Read more about the rules design pattern at Michael Whelan's [article](http://www.michael-whelan.net/rules-design-pattern/).

```javascript

@Injectable()
export class RenderingTranslationService {

    constructor() {
    }

    // Rendering rules goes here...
    private renderingTranslationRules: RenderingTranslationRule[] = [
        RenderingTranslationRule.freetextRenderingRule,
        RenderingTranslationRule.radioRenderingRule,
        RenderingTranslationRule.checkboxRenderingRule,
        RenderingTranslationRule.comboboxRenderingRule
    ];

    getRenderingForQuestion(question: Question): Rendering {

        let renderingToReturn;
        for (const renderingTranslationRule of this.renderingTranslationRules) {
            if (renderingTranslationRule.booleanExp(question)) {
                renderingToReturn = renderingTranslationRule.rendering;
                break;
            }
        }

        if (!renderingToReturn) {
            throw new Error('No renderings for question: ' + question.externalQuestionId);
        }

        return renderingToReturn;
    }
}

```

The rules are simply added to renderingTranslationRules array which be enumerated and extract the rendering from the RenderingTranslationRule, which first matches the rendering expression (again, enum object type). More rules can be added as long as they have a Rendering and booleanExp. Watch how the RenderingTranslationRules are implemented [here](https://github.com/lydemann/questionnaire-render-demo/blob/master/src/app/question/rendering-translation.service.ts).


## Showing the rendering type

Lastly, after the mapping is done, we now have a rendering type to show in question template. The supported rendering types are *TEXTBOX*, *RADIO*, *COMBOBOX* and *CHECKBOX* and they are extracted using the above *getRenderingForQuestion* method. As the application scales, I would recommend to extract these rendering in seperate presentation component for keeping the question template as slim as possible.

The question html template can be viewed [here](https://github.com/lydemann/questionnaire-render-demo/blob/master/src/app/question/question.component.html)

# Putting it all together

There you have it. The questionnaire was rendered using these steps:

* Mapping the validationRules to the formGroup
* Creating the formGroup using the mapped validationRules and questionnaire
* Mapping the ui rendering rules from the question
* Showing the right question rendering in the question template using a switch


The complete solution can be found at my github [here](https://github.com/lydemann/questionnaire-render-demo)
