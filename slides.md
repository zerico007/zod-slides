---
theme: seriph

background: "/static/bitovi_bg.png"
# some information about your slides (markdown enabled)
title: Schema Validation in Forms with Zod

class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Schema Validation in Forms with Zod

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Let's begin <carbon:arrow-right />
</div>

<div class="abs-bl flex gap-4 m-6">
  <div class="flex flex-col items-center">
    <img src="/static/bavin_edwards.jpg" class="rounded-full w-32 h-32 object-contain" />
    <p class="mt-2 text-sm text-white">Bavin Edwards</p>
  </div>
  <div class="flex flex-col items-center">
    <img src="/static/ali_shouman.jpg" class="rounded-full w-32 h-32 object-contain" />
    <p class="mt-2 text-sm text-white">Ali Shouman</p>
  </div>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: fade-out
---

# What is Zod?

TypeScript-first schema validation with static type inference.

- Zero external dependencies
- Works in Node.js and all modern browsers
- Tiny: 2kb core bundle (gzipped)
- Immutable API: methods return a new instance
- Concise interface
- Works with TypeScript and plain JS
- Built-in JSON Schema conversion
- Extensive ecosystem
- Zod is pretty much like Typescript written as functional programming, making it a powerful tool for building robust applications.
  <br>
  <br>

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

<!--
Here is another comment.
-->

---
transition: slide-up
level: 2
---

# How do we define a schema?

```ts {all|1|3-10}{lines:true,startLine:1,maxHeight:'500px'}
import { z } from 'zod';

const userSchema = z.object({
  name: z.string(),
  age: z.number(),
  email: z.email({ 
    error: "Invalid email address",
    pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ 
  }),
});
```

---
transition: slide-up
level: 2
---

# Let's look at a more complex example

```ts {all|4-23|26-34|35|36-44|47}{lines:true,startLine:1,maxHeight:'500px'}
import startCase from "lodash.startcase";
import { z } from "zod";

export const states = ["AL", "AK", "AZ", "AR", ...] as const;

export const genders = ["male", "female"] as const;

const maritalStatuses = ["single", "married", "divorced", "widowed"] as const;

export const stateOptions = states.map((state) => ({
  value: state,
  label: state,
}));

export const genderOptions = genders.map((gender) => ({
  value: gender,
  label: startCase(gender),
}));

export const maritalStatusOptions = maritalStatuses.map((status) => ({
  value: status,
  label: startCase(status),
}));

export const personalInfoFormSchema = z.object({
  investmentAmount: z.preprocess(
    (val) => (typeof val === "string" && Boolean(val) ? parseFloat(val) : val),
    z.coerce
      .number()
      .min(1000, { message: "Investment amount must be greater than $1000" })
      .max(2_000_000, {
        message: "Investment amount must be less than $2,000,000",
      })
  ),
  stateOfResidence: z.enum(states),
  dateOfBirth: z.preprocess(
    (val) => (typeof val === "string" ? new Date(val) : val),
    z
      .date()
      .min(new Date("1900-01-01"), { message: "Please select a valid date" })
      .max(new Date(), { message: "Please select a valid date" })
  ),
  gender: z.enum(genders),
  maritalStatus: z.enum(maritalStatuses),
});

export type PersonalInfoFormType = z.infer<typeof personalInfoFormSchema>;
```

---
transition: fade-out
level: 2
---

# Some additional features - SuperRefine

```ts{all|3-6|9-30|31-44|32|33|34|35-39|45}{lines:true,startLine:1,maxHeight:'500px'}
import { z } from "zod";

const jointPersonalInfoErrorMessages = {
  jointBirthdate: "Spouse birthdate is required for joint life annuity type",
  jointGender: "Spouse gender is required for joint life annuity type",
};

const spiaInfoFormSchema = z
  .object({
    payoutType: z.enum(payoutTypes).default("single"),
    paymentsStartDate: z.preprocess(
      (val) => (typeof val === "string" ? new Date(val) : val),
      z
        .date()
        .min(getDateMonthAway(), { message: "Please select a valid date" })
        .max(getDateYearAway(), { message: "Please select a valid date" })
        .default(getDateMonthAway()),
    ),
    incomePayoutFrequency: z.enum(incomePayoutFrequencies).default("monthly"),
    jointBirthdate: z.optional(
      z.preprocess(
        (value) => (typeof value === "string" ? new Date(value) : value),
        z
          .date()
          .min(new Date("1900-01-01"), { message: "Please select a valid date" })
          .max(new Date(), { message: "Please select a valid date" }),
      ),
    ),
    jointGender: z.optional(z.enum(genders)),
  })
  .superRefine((value, ctx) => {
    if (value.payoutType === "joint-life") {
      for (const [key, message] of Object.entries(jointPersonalInfoErrorMessages)) {
        if (value[key as keyof typeof jointPersonalInfoErrorMessages] === undefined) {
          ctx.addIssue({
            code: z.ZodIssueCode.custom,
            message,
            path: [key],
            fatal: true,
          });
        }
      }
    }
  });
export type SpiaInfoFormType = z.infer<typeof spiaInfoFormSchema>;
```

---
transition: slide-up
level: 2
---

# Some Typescript-inspired features

```ts {all|3-5|7-9|11-16|18-39}{lines:true,startLine:1,maxHeight:'500px'}
import { z } from "zod";

const fiaFieldsSchema = fiaInfoFormSchema.pick({
  investmentPeriod: true,
});

const ratingsSchemaField = spiaFiltersFormSchema.innerType().pick({
  rating: true,
}).required();

const personalFieldsSchema = personalInfoFormSchema.pick({
  stateOfResidence: true,
  investmentAmount: true,
}).partial({
  stateOfResidence: true,
});

const fiaFiltersFormSchema = personalFieldsSchema
  .merge(fiaFieldsSchema)
  .merge(ratingsSchemaField)
  .extend({
    institutionId: z.preprocess(
      (val) => {
        if (typeof val === "string") {
          const parsed = parseInt(val);
          return isNaN(parsed) ? undefined : parsed;
        }
        return val;
      },
      z.coerce.number().optional(),
    ),
    age: z.preprocess(
      (val) => (typeof val === "string" && Boolean(val) ? parseInt(val) : val),
      z.coerce
        .number()
        .min(25, { message: "Age must be 25 or greater" })
        .max(99, { message: "Age must be 99 or less" }),
    ),
  });

export type FiaFiltersFormType = z.infer<typeof fiaFiltersFormSchema>;
```

---
transition: slide-up
level: 2
---
# Using Zod with React Hook Form

```ts {all|1-8|12|14-22|24-28|30-34|37-62|65}{lines:true,startLine:1,maxHeight:'500px'}
import type { SpiaInfoFormType } from "@/lib/schemas/spia-info-form-schema";

import { zodResolver } from "@hookform/resolvers/zod";
import { useRouter } from "next/navigation";
import { useEffect, useMemo } from "react";
import { useForm } from "react-hook-form";

import { spiaInfoFormSchema } from "@/lib/schemas/spia-info-form-schema";

export const useSpiaInfoForm = () => {
  const router = useRouter();
  const { updater, getParamValue, getNextUrl } = useUpdateSearchParams();

  const parsedForm = useMemo(() => {
    return spiaInfoFormSchema.safeParse({
      payoutType: getParamValue("payoutType"),
      incomePayoutFrequency: getParamValue("incomePayoutFrequency"),
      paymentsStartDate: getParamValue("paymentsStartDate", (value) => new Date(value)),
      jointBirthdate: getParamValue("jointBirthdate", (value) => new Date(value)),
      jointGender: getParamValue("jointGender"),
    });
  }, [getParamValue]);

  useEffect(() => {
    if (!parsedForm.success) {
      router.push("/onboarding");
    }
  }, [parsedForm.success, router]);

  const form = useForm<SpiaInfoFormType>({
    resolver: zodResolver(spiaInfoFormSchema),
    mode: "onBlur",
    defaultValues: parsedForm.success ? parsedForm.data : undefined,
  });

  function onSubmit(values: SpiaInfoFormType) {
    updater(
      {
        payoutType: values.payoutType,
        incomePayoutFrequency: values.incomePayoutFrequency,
        paymentsStartDate: values.paymentsStartDate.toISOString(),
        jointBirthdate: values.jointBirthdate?.toISOString() ?? "",
        jointGender: values.jointGender ?? "",
        // smart defaults that will be updatable as filters on results page
        jointType: "non-reducing",
        jointPayoutPercentage: "100",
        primaryPayoutPercentage: "100",
        guaranteeType: "life-with-period-certain",
      },
      (params) => {
        if (values.payoutType !== "joint-life") {
          removeParams(params, [
            "jointBirthdate",
            "jointGender",
            "jointType",
            "jointPayoutPercentage",
            "primaryPayoutPercentage",
          ]);
        }
      },
    );
    router.push(getNextUrl("/onboarding/results"));
  }

  return { form, onSubmit, getNextUrl, isFormValid: parsedForm.success };
};
```

---
transition: slide-up
level: 2
---
# And finally...

Let's look at some real client code to see a form using zod in action