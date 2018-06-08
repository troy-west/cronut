# Cronut: Scheduled Execution via Quartz and Integrant

[![Clojars Project](https://img.shields.io/clojars/v/com.troy-west/cronut.svg)](https://clojars.org/com.troy-west/cronut)

> "Cronut is a good name, you can call it that if you want" - James Sofra

Cronut provides a data-first Clojure wrapper for the [Quartz Job Scheduler](http://www.quartz-scheduler.org/)

# Summary

Quartz is richly featured, open source jobn scheduling library that is fairly standard on the JVM.

Clojure has two wrappers for Quartz already:

1. [Quartzite](https://github.com/michaelklishin/quartzite) by Michael Klishin / ClojureWerkz
2. [Twarc](https://github.com/prepor/twarc) by Andrew Rudenko / Rudenko

How does Cronut differ?

1. Configured entirely from data (with [Integrant](https://github.com/weavejester/integrant) bindings provided)
2. No macros or new protocols, just implement the org.quartz.Job interface
3. No global Clojure state
4. Latest version of Quartz
5. Tagged literals to shortcut common use-cases (#cronut/cron, #cronut/interval)
6. Easily extensible for further triggers / tagged literals

# Usage

## :cronut/scheduler

Cronut provides lifecycle implementation for the Quartz Scheduler, exposed via Integrant bindings.

The scheduler supports two fields:

1. (optional) time-zone: default for the scheduler, where triggers support it. e.g. "Australia/Melbourne"
2. (required) schedule: a sequence of 'items' to schedule, each being a map containing a :job and :trigger

e.g

````clojure
:cronut/scheduler {:time-zone "Australia/Melbourne"
                   :schedule  [{:job     #ig/ref :test.job/two
                                :trigger #cronut/interval 3500}

                               {:job     #ig/ref :test.job/two
                                :trigger #cronut/cron "*/8 * * * * ?"}]}}
````

### The :job

The :job in every scheduled item must implement the org.quartz.Job interface

The expectation being that every 'job' in your Integrant system will reify that interface, either directly via `reify`
or by returning a defrecord that implements the interface. e.g.

````clojure
(defmethod ig/init-key :test.job/one
  [_ config]
  (reify Job
    (execute [this job-context]
      (log/info "Reified Impl:" config))))

(defrecord TestDefrecordJobImpl [identity description recover? durable? test-dep]
  Job
  (execute [this job-context]
    (log/info "Defrecord Impl:" this)))

(defmethod ig/init-key :test.job/two
  [_ config]
  (map->TestDefrecordJobImpl config))
````

Cronut supports further Quartz configuration of jobs (identity, description, recovery, and priority) by expecting those
values to be assoc'd onto your job. You do not have to set them (in fact in most cases you can likely ignore them),
however if you do want that control you will likely use the defrecord approach as opposed to the simpler reify option
and pass that configuration through edn, e.g.

````clojure
:test.job/two     {:identity    ["job-two" "test"]
                   :description "test job"
                   :recover?    true
                   :durable?    false
                   :dep-one     #ig/ref :dep/one
                   :dep-two     #ig/ref :test.job/one}
````                    

### The :trigger

The :trigger in every scheduled item must resolve to an org.quartz.Trigger of some variety or another, to ease that 
resolution Cronut provides the following tagged literals:

### Tagged Literals

#### #cronut/cron: Simple Cron Scheduling

A job is scheduled to run on a cron by using the #cronut/cron tagged literal followed by a valid cron expression

The job will start immediately when the system is initialized

````clojure
:trigger #cronut/cron "*/8 * * * * ?"
````

#### #cronut/interval: Simple Interval Scheduling

A job is scheduled to run periodically by using the #cronut/interval tagged literal followed by a milliseconds value 

````clojure
:trigger #cronut/interval 3500
````

#### #cronut/trigger: Full (and extensible) Trigger Definition

Both #cronut/cron and #cronut/interval are effectively shortcuts to full trigger definition with sensible defaults.

The #cronut/trigger tagged literal supports the full set of Quartz configuration for Simple and Cron triggers:

````clojure
:trigger #cronut/trigger {:type        :simple
                          :interval    3000
                          :repeat      :forever
                          :identity    ["trigger-two" "test"]
                          :description "test trigger"
                          :start       #inst "2019-01-01T00:00:00.000-00:00"
                          :end         #inst "2019-02-01T00:00:00.000-00:00"
                          :priority    5}
````

This tagged literal calls a Clojure multi-method that you can extend with your own config -> trigger implementation

## Integrant

When initializing an Integrant system you will need to provide the Cronut tagged literals.

See: troy-west.cronut/init-system for a convenience implementation if you prefer:

````clojure
(def data-readers
  {'ig/ref          ig/ref
   'cronut/trigger  troy-west.cronut/trigger-builder
   'cronut/cron     troy-west.cronut/shortcut-cron
   'cronut/interval troy-west.cronut/shortcut-interval})

(defn init-system
  "Convenience for starting integrant systems with cronut data-readers"
  ([config]
   (init-system config nil))
  ([config readers]
   (ig/init (edn/read-string {:readers (merge data-readers readers)} config))))
````

## Example System

Given a simple Integrant configuration of two jobs and four triggers.

Job Two is executed on multiple schedules as defined by the latter three triggers. 

````clojure
{:test.job/one     {}

 :test.job/two     {:identity    ["job-two" "test"]
                    :description "test job"
                    :recover?    true
                    :durable?    false
                    :dep-two     #ig/ref :test.job/one}

 :cronut/scheduler {:time-zone "Australia/Melbourne"
                    :schedule  [;; basic interval
                                {:job     #ig/ref :test.job/one
                                 :trigger #cronut/trigger {:type      :simple
                                                           :interval  2
                                                           :time-unit :seconds
                                                           :repeat    :forever}}
                             
                                ;; shortcut interval via cronut/interval data-reader
                                {:job     #ig/ref :test.job/two
                                 :trigger #cronut/interval 3500}

                                ;; basic cron
                                {:job     #ig/ref :test.job/two
                                 :trigger #cronut/trigger {:type :cron
                                                           :cron "*/4 * * * * ?"}}
                             
                                ;; shortcut cron via cronut/cron data-reader
                                {:job     #ig/ref :test.job/two
                                 :trigger #cronut/cron "*/8 * * * * ?"}]}}

````

And the associated Integrant lifecycle impl, note:

- test.job/one reifies the org.quartz.Job interface
- test.job/two instantiates a defrecord (that allows some further quartz job configuration)  

````clojure
(defmethod ig/init-key :test.job/one
  [_ config]
  (reify Job
    (execute [this job-context]
      (log/info "Reified Impl:" config))))

(defrecord TestDefrecordJobImpl [identity description recover? durable? test-dep]
  Job
  (execute [this job-context]
    (log/info "Defrecord Impl:" this)))

(defmethod ig/init-key :test.job/two
  [_ config]
  (map->TestDefrecordJobImpl config))
````

We can realise that system and run those jobs (See troy-west.cronut.integration-fixture for full example):

````clojure
(require '[troy-west.cronut.integration-fixture :as itf])
=> nil

(itf/initialize!)
=>
{:dep/one {:a 1},
 :test.job/one #object[troy_west.cronut.integration_fixture$eval2343$fn$reify__2345
                       0x2e906b8a
                       "troy_west.cronut.integration_fixture$eval2343$fn$reify__2345@2e906b8a"],
 :test.job/two #troy_west.cronut.integration_fixture.TestDefrecordJobImpl{:identity ["job-two" "test"],
                                                                          :description "test job",
                                                                          :recover? true,
                                                                          :durable? false,
                                                                          :test-dep nil,
                                                                          :dep-one {:a 1},
                                                                          :dep-two #object[troy_west.cronut.integration_fixture$eval2343$fn$reify__2345
                                                                                           0x2e906b8a
                                                                                           "troy_west.cronut.integration_fixture$eval2343$fn$reify__2345@2e906b8a"]},
 :cronut/scheduler #object[org.quartz.impl.StdScheduler 0x7565dd8e "org.quartz.impl.StdScheduler@7565dd8e"]}
 
 (itf/shutdown!)
=> nil 
````

## License

Copyright © 2018 [Troy-West, Pty Ltd.](http://www.troywest.com)

Distributed under the Eclipse Public License either version 1.0 or (at your option) any later version.