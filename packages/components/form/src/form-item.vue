<script setup lang="ts">
  import {
    computed,
    inject,
    nextTick,
    onBeforeUnmount,
    onMounted,
    provide,
    ref,
    toRefs,
    useSlots,
  } from 'vue';
  import { castArray, get, set, cloneDeep, isEqual } from 'lodash';
  import Schema from 'async-validator';
  import type { CSSProperties } from 'vue';
  import type { RuleItem } from 'async-validator';
  import { addUnit, isArray, isString, isUndefined, createNamespace } from '../../../utils';
  import { formContextKey, formItemContextKey } from './context';
  import { FormItemProps } from './form-item';
  import type { FormItemContext, FormItemRule } from './form';
  import { useSize } from './use-form';

  const props = defineProps(FormItemProps);

  const { createBEM } = createNamespace('form-item');

  const slots = useSlots();

  const size = useSize();

  let initialValues = undefined;

  const validateState = ref<'' | 'error' | 'validating' | 'success'>('');
  const validateMessage = ref('');

  const formContext = inject(formContextKey, undefined);

  const classNames = computed(() => [
    createBEM(),
    createBEM(`--${size.value}`),
    {
      [createBEM('--required')]: isRequired.value,
      [createBEM('--error')]: validateState.value === 'error',
    },
  ]);

  const labelStyles = computed<CSSProperties>(() => {
    if (formContext?.labelPosition.value === 'top') {
      return {};
    }
    const labelWidth = addUnit(props.labelWidth || formContext?.labelWidth.value || '');
    if (labelWidth) {
      return {
        width: labelWidth,
      };
    }
    return {};
  });

  const contentStyles = computed<CSSProperties>(() => {
    if (formContext?.labelPosition.value === 'top' || formContext?.inline.value) {
      return {};
    }
    if (!props.label && !slots.label) {
      const labelWidth = addUnit(props.labelWidth || formContext?.labelWidth.value || '');
      if (labelWidth) {
        return {
          marginLeft: labelWidth,
        };
      }
    }
    return {};
  });

  const propString = computed(() => {
    if (!props.prop) return '';
    return isString(props.prop) ? props.prop : props.prop.join('.');
  });

  const fieldValue = computed(() => {
    const model = formContext?.model?.value;
    if (!model || !props.prop) {
      return;
    }
    return get(model, props.prop);
  });

  // props.rules + FormContext.rules + required rules
  const allRules = computed(() => {
    const rules: FormItemRule[] = props.rules ? castArray(props.rules) : [];

    const formRules = formContext?.rules?.value;
    if (formRules && props.prop) {
      const _rules = get(formRules, props.prop);

      if (_rules) {
        rules.push(...castArray(_rules));
      }
    }

    if (!isUndefined(props.required)) {
      rules.push({ required: !!props.required });
    }

    return rules;
  });

  const isRequired = computed(() => allRules.value.some((rule) => rule.required === true));

  const getFilteredRules: (trigger: string) => RuleItem[] = (trigger) => {
    const rules = allRules.value;

    return (
      rules
        .filter((rule) => {
          if (!rule.trigger || !trigger) return true;
          if (isArray(rule.trigger)) {
            return rule.trigger.includes(trigger);
          } else {
            return rule.trigger === trigger;
          }
        })
        // eslint-disable-next-line
        .map(({ trigger, ...rule }) => rule)
    );
  };

  const validate: FormItemContext['validate'] = (trigger, callback) => {
    const rules = getFilteredRules(trigger);

    if (rules.length === 0) {
      callback?.(true);
      return;
    }

    validateState.value = 'validating';

    const modelName = propString.value;
    const validator = new Schema({
      [modelName]: rules,
    });
    return validator.validate(
      { [modelName]: fieldValue.value },
      { firstFields: true },
      (errors, fields) => {
        if (errors) {
          validateState.value = 'error';
          validateMessage.value = errors[0].message as string;
          callback?.(false, fields);
        } else {
          validateState.value = 'success';
          validateMessage.value = '';
          callback?.(true);
        }
      },
    );
  };

  const resetField: FormItemContext['resetField'] = async () => {
    const model = formContext?.model?.value;
    if (!model || !props.prop) return;
    const fieldValue = get(model, props.prop);

    if (!isEqual(fieldValue, initialValues)) {
      set(model, props.prop, cloneDeep(initialValues));
    }

    await nextTick();
    validateState.value = '';
    validateMessage.value = '';
  };

  onMounted(() => {
    if (props.prop) {
      formContext?.addField(context);
      initialValues = cloneDeep(fieldValue.value);
    }
  });

  onBeforeUnmount(() => {
    formContext?.removeField(context);
  });

  const context: FormItemContext = {
    ...toRefs(props),
    validate,
    resetField,
  };

  // 为其它子组件提供依赖，方便调用表单的方法，如校验方法。
  provide(formItemContextKey, context);
</script>

<template>
  <div :class="classNames">
    <!-- label -->
    <label v-if="label || slots.label" :class="createBEM('label')" :style="labelStyles">
      <slot name="label">{{ label }}</slot>
    </label>
    <!-- content -->
    <div :class="createBEM('content')" :style="contentStyles">
      <slot></slot>
      <!-- error-tip -->
      <transition name="fade">
        <div v-if="validateState === 'error'" :class="createBEM('--error-tip')">{{
          validateMessage
        }}</div>
      </transition>
    </div>
  </div>
</template>
